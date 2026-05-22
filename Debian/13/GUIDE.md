# VPS Hardening Guide (Debian-13)

| &nbsp; | &nbsp; |
| --- | --- |
| **Target OS** | Debian 13.x (Trixie - current "stable") |
| **Guide Version** | Debian-13-Rev-0 |

## Contextual Preamble

### Prerequisites

- Root SSH access to the VPS
- VPS provider snapshot capability (take one before starting)
- An SSH key pair ready on your local machine (or generate one in Section 3)

### Out of Scope

- Partition restructuring on live systems
- Service-specific hardening (Nginx, Postfix, MariaDB, etc.)
- Container / Docker hardening
- Cloud provider security group configuration
- Multi-user access policies
- Port knocking (noted as optional escalation in Section 3, not prescribed)


## Section Order

| # | Section |
|---|---------|
| 1 | [Operating System Baseline](#section-1--operating-system-baseline) |
| 2 | [User & Authentication](#section-2--user--authentication) |
| 3 | [Secure Shell (SSH)](#section-3--secure-shell-ssh) |
| 4 | [Firewall](#section-4--firewall) |
| 5 | [Package Integrity](#section-5--package-integrity) |
| 6 | [Mandatory Access Control (MAC)](#section-6--mandatory-access-control-mac) |
| 7 | [Logging & Audit](#section-7--logging--audit) |
| 8 | [Kernel Hardening](#section-8--kernel-hardening) |
| 9 | [Filesystem Hardening](#section-9--filesystem-hardening) |
| 10 | [Performance Optimization](#section-10--performance-optimization) |
| 11 | [Intrusion Detection System](#section-11--intrusion-detection-system) |
| 12 | [Maintenance Hygiene](#section-12--maintenance-hygiene) |

---

## Section 1 — Operating System Baseline

> Goal: Start from the leanest, most up-to-date system state possible. Every unnecessary package or service is an unmonitored attack surface.

### Checklist

- [ ] **1.1** Set correct hostname
- [ ] **1.2** Set correct timezone
- [ ] **1.3** Configure local Indonesian mirror (`kartolo.sby.datautama.net.id`)
- [ ] **1.4** Verify APT GPG verification is active, then apply all pending updates
- [ ] **1.5** Install essential baseline packages
- [ ] **1.6** Configure `unattended-upgrades` for security-only auto-updates
- [ ] **1.7** Remove known bloat packages
- [ ] **1.8** Disable and stop unused services
- [ ] **1.9** Verify `systemd-timesyncd` is running and synchronized

### Rationale

**1.1 — Hostname**  
A meaningful hostname prevents confusion in logs. Debian 13 defaults to `debian` — every log entry will be ambiguous if you ever cross-reference across machines.

**1.2 — Timezone**  
Set to your actual operational timezone. Mismatched timezones corrupt log correlation. `UTC` is acceptable and often preferred for servers managed across regions.

**1.3 — Local Indonesian mirror (`kartolo.sby.datautama.net.id`)**  
`kartolo.sby.datautama.net.id` is operated by PT Data Utama, Surabaya, and is an official Debian mirror listed on `debian.org/mirror/list`. It covers the Debian 13 (trixie) main archive, security updates, and point-release updates over HTTPS. Configuring this mirror before any `apt` operation ensures all package downloads route domestically. All packages remain GPG-verified against Debian's archive signing key regardless of which mirror delivers them — using a local mirror does not weaken APT's verification chain.

Debian 13 uses DEB822-format sources in `/etc/apt/sources.list.d/debian.sources` alongside the traditional `/etc/apt/sources.list` — both files are active and processed by APT. The guide targets `debian.sources` for the mirror configuration since it is the DEB822 source. Both files are backed up before modification so the fallback can fully restore the pre-kartolo state.

**1.4 — GPG verification + full upgrade**  
APT enforces GPG signature verification by default — every package is checked against Debian's archive signing key before installation. Unlike `dnf`'s per-repo `gpgcheck` flag, APT's verification is archive-wide and cannot be silently disabled without explicit overrides like `[trusted=yes]` in sources or `--allow-unauthenticated` flags. This step confirms no such bypass is present, then runs a full upgrade. Hardening built on top of unpatched packages is built on a weak foundation. Reboot afterward if the kernel was upgraded.

**1.5 — Install essential packages**  
A minimal Debian 13 VPS image does not ship with `nano`, `zsh`, `firewalld`, `fail2ban`, `auditd`, `aide`, `debsums`, `apparmor-utils`, or `libpam-pwquality`. On Debian, `auditd` is a single package includes the daemon and all CLI tools (`auditctl`, `ausearch`, `aureport`, `augenrules`). Installing everything the guide depends on upfront in one pass makes the guide fully script-friendly.

**1.6 — `unattended-upgrades` (security only)**  
`unattended-upgrades` is Debian's equivalent of `dnf-automatic`. It applies security patches automatically without pulling in feature updates. Configured via `/etc/apt/apt.conf.d/50unattended-upgrades` and triggered by the `apt-daily-upgrade.timer` systemd unit. The `Allowed-Origins` list is restricted to `trixie-security` only — non-security updates are not auto-applied.

**1.7 — Package removal**  
Packages with no purpose are packages that can have vulnerabilities. Common targets: `telnet`, `rsh-client`, `nis`, `rpcbind` (if not using NFS), `avahi-daemon`, `cups`. Debian minimal images are lean — verify what's present before removing.

**1.8 — Unused services**  
A stopped and disabled service cannot be exploited. Audit with `systemctl list-units --type=service --state=running` and be deliberate about everything that's active.

**1.9 — `systemd-timesyncd`**  
Debian 13 uses `systemd-timesyncd` for NTP synchronization by default — it is lighter than `chrony` and appropriate for general VPS use. Accurate time is not optional: TLS certificate validation, `auditd` timestamps, and `fail2ban` log parsing all depend on it.

### Scriptable Commands

```bash
# --- 1.1 Set hostname ---
hostnamectl set-hostname your-hostname
# Verify:
hostnamectl status

# --- 1.2 Set timezone ---
timedatectl set-timezone Asia/Jakarta  # adjust to your zone
# Verify:
timedatectl status

# --- 1.3 Configure local Indonesian mirror ---
# Mirror: kartolo.sby.datautama.net.id (PT Data Utama, Surabaya)
# Official Debian mirror — listed at debian.org/mirror/list
# Covers: trixie main archive, security, updates
# Protocol: HTTPS

# Test mirror reachability before committing:
curl -o /dev/null -s -w "HTTP %{http_code} — Time: %{time_total}s\n" \
  https://kartolo.sby.datautama.net.id/debian/dists/trixie/Release
# Expected: HTTP 200 — Time: <1s
# If unreachable, skip this step and proceed with default deb.debian.org

# Backup both source files (traditional and DEB822-format):
cp /etc/apt/sources.list~
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cp /etc/apt/sources.list.d/debian.sources /etc/apt/sources.list.d/debian.sources.bak
# Note any additional provider sources:
ls /etc/apt/sources.list.d/

cat > /etc/apt/sources.list.d/debian.sources << 'EOF'
# Source Mirror: PT Data Utama, Surabaya (official Debian mirror)

# Base & Updates
Types: deb deb-src
URIs: https://kartolo.sby.datautama.net.id/debian
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Architectures: amd64
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

# Security
Types: deb deb-src
URIs: https://kartolo.sby.datautama.net.id/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Architectures: amd64
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF

# Update metadata and verify mirror is working:
apt update
# All repos should resolve against kartolo.sby.datautama.net.id

# --- 1.4 Verify GPG enforcement + full upgrade ---
# Confirm no source has a [trusted=yes] bypass:
grep -rE 'trusted=yes|allow-unauthenticated|AllowUnauthenticated' \
  /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
# Expected: no output — any result here is a security concern

# Confirm Debian archive signing key is present:
ls /etc/apt/trusted.gpg.d/
# Expected: debian-archive-*.gpg files

# Full upgrade (downloads from kartolo.sby.datautama.net.id):
apt upgrade -y
# Reboot if kernel was upgraded:
# reboot

# --- 1.5 Install essential packages ---
apt install -y \
  nano \
  zsh \
  curl \
  firewalld \
  fail2ban \
  auditd \
  aide \
  debsums \
  apparmor-utils \
  libpam-pwquality

# --- 1.6 Configure unattended-upgrades (security-only) ---
apt install -y unattended-upgrades

cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Dependencies "false";
EOF

cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF

systemctl enable --now unattended-upgrades
# Verify:
systemctl status unattended-upgrades
systemctl status apt-daily-upgrade.timer

# --- 1.7 Remove bloat packages ---
apt purge -y telnet rsh-client nis rpcbind avahi-daemon cups 2>/dev/null || true
apt autoremove -y

# --- 1.8 Audit running services (manual review step) ---
systemctl list-units --type=service --state=running
# Disable anything not intentionally needed:
# systemctl disable --now <service-name>

# --- 1.9 Verify systemd-timesyncd ---
systemctl enable --now systemd-timesyncd
timedatectl status
# Confirm: "NTP service: active" and "System clock synchronized: yes"
```

> **Fallback:** If `kartolo.sby.datautama.net.id` is unreachable on first run, restore both source files and proceed with the Debian CDN:
> ```bash
> cp /etc/apt/sources.list.bak /etc/apt/sources.list
> cp /etc/apt/sources.list.d/debian.sources.bak /etc/apt/sources.list.d/debian.sources
> apt update
> ```

### [Back to Top](#section-order)

---

## Section 2 — User & Authentication

> Goal: Eliminate root as an operable account. All human access flows through `dan` with a password and sudo. Root becomes a locked, inert account.

### Checklist

- [ ] **2.1** Create user `dan` with a strong password
- [ ] **2.2** Add `dan` to the `sudo` group
- [ ] **2.3** Verify `sudo` group is enabled in `sudoers`
- [ ] **2.4** Set secure `umask` system-wide and for `dan`
- [ ] **2.5** Set `zsh` as default shell for `dan` and `root`
- [ ] **2.6** Install oh-my-zsh for `root`
- [ ] **2.7** Install oh-my-zsh for `dan`
- [ ] **2.8** Configure `fino-time` theme and `umask` in `.zshrc` for both users
- [ ] **2.9** Lock the root account
- [ ] **2.10** Harden PAM — limit login attempts via `pam_faillock`
- [ ] **2.11** Enforce password quality policy via `pwquality`

### Rationale

**2.1 & 2.2 — Create `dan` and add to `sudo`**  
The VPS arrives with only a root account. Before locking root (2.9), `dan` must exist and be fully operational — including sudo access. **Do not lock root before verifying `dan` can sudo.** Doing it in the wrong order will lock you out of your own server.

On Debian, the privilege escalation group is `sudo`, not `wheel`. The distinction is in naming only — the mechanism is identical.

**2.3 — Verify `sudo` in sudoers**  
Debian enables the `sudo` group in `/etc/sudoers` by default: `%sudo ALL=(ALL:ALL) ALL`. Confirm this line is uncommented and active. We use the `sudo` group rather than a direct `sudoers` entry for `dan` — it's cleaner and easier to audit.

**2.4 — `umask 027`**  
The default `umask 022` creates files readable by all users. `027` restricts new files to owner read/write, group read-only, and no access for others. Applied system-wide via `/etc/profile` — this covers both bash and zsh login shells. Per-user `.zshrc` entries are added in step 2.8 to also cover interactive non-login zsh sessions.

**2.5 — Set zsh as default shell**  
`chsh -s /bin/zsh` updates `/etc/passwd` to set zsh as the login shell for both `dan` and `root`. Root's shell is set here even though the account is locked in step 2.9 — when you access root via `sudo -i` or `sudo su` from `dan`, you get root's configured shell.

**2.6 & 2.7 — Install oh-my-zsh**  
oh-my-zsh is installed via its official `install.sh` script fetched over HTTPS from GitHub. The `--unattended` flag suppresses interactive prompts. We install for `root` first while operating as root, then for `dan` using `su - dan` to ensure oh-my-zsh is correctly owned by and configured for each user separately.

**2.8 — Configure `fino-time` theme and `umask`**  
`fino-time` is a built-in oh-my-zsh theme — no additional download needed. We set it by replacing the default `ZSH_THEME` value in each user's `.zshrc`, and append `umask 027` to cover interactive non-login sessions where `/etc/profile` is not sourced.

**2.9 — Lock root (`passwd -l root`)**  
Once `dan` is confirmed working with sudo, root should be completely inert. `passwd -l` places a `!` prefix on the password hash in `/etc/shadow` — no authentication method can unlock it without going through `dan` first.

**2.10 — PAM `pam_faillock`**  
`pam_faillock` is part of `libpam-modules`, which is a core Debian package — no additional install needed. Tracks failed authentication attempts and locks an account after a threshold. Configuration: 5 failed attempts → 15 minute lockout.

On Debian, PAM is managed through `pam-auth-update` which generates `/etc/pam.d/common-*` files from profiles in `/usr/share/pam-configs/`. Debian includes a `faillock` pam-config profile in `libpam-modules`. If it was applied during installation, the faillock lines will already be in `common-auth` and `common-account`. This step checks first and adds manually only if missing — important because manually added PAM lines can be overwritten by a future `pam-auth-update --force` run.

**2.11 — `pwquality`**  
`libpam-pwquality` was installed in Section 1.5. Configuration via `/etc/security/pwquality.conf`. Minimum length 12, require mixed character classes, reject passwords too similar to the username.

### Scriptable Commands

```bash
# --- 2.1 Create user dan with password ---
useradd -m -s /bin/bash dan
passwd dan
# You will be prompted to enter and confirm a password interactively

# --- 2.2 Add dan to sudo group ---
usermod -aG sudo dan

# --- 2.3 Verify sudo group is active in sudoers ---
grep -E '^\s*%sudo' /etc/sudoers
# Expected output: %sudo   ALL=(ALL:ALL)    ALL
# If commented out, uncomment it (never edit /etc/sudoers directly):
# visudo

# --- 2.4 Set umask 027 system-wide ---
echo 'umask 027' >> /etc/profile
# Covers all login shells (bash and zsh)
# Per-user .zshrc entries added in step 2.8 after oh-my-zsh creates the files

# --- 2.5 Set zsh as default shell for dan and root ---
chsh -s /bin/zsh root
chsh -s /bin/zsh dan
# Verify:
grep -E '^(root|dan):' /etc/passwd | cut -d: -f1,7
# Expected: root:/bin/zsh and dan:/bin/zsh

# --- 2.6 Install oh-my-zsh for root ---
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" \
  "" --unattended
# Installs to /root/.oh-my-zsh and creates /root/.zshrc
# Verify:
ls /root/.oh-my-zsh/

# --- 2.7 Install oh-my-zsh for dan ---
su - dan
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" \
  "" --unattended
exit
# Installs to /home/dan/.oh-my-zsh and creates /home/dan/.zshrc
# Verify:
ls /home/dan/.oh-my-zsh/

# --- 2.8 Configure fino-time theme and umask for both users ---
# Root:
sed -i 's/^ZSH_THEME=.*/ZSH_THEME="fino-time"/' /root/.zshrc
echo 'umask 027' >> /root/.zshrc

# Dan:
sed -i 's/^ZSH_THEME=.*/ZSH_THEME="fino-time"/' /home/dan/.zshrc
echo 'umask 027' >> /home/dan/.zshrc

# Verify:
grep 'ZSH_THEME' /root/.zshrc /home/dan/.zshrc
# Expected: ZSH_THEME="fino-time" in both files

# --- VERIFY dan can sudo BEFORE locking root ---
# Open a second SSH session and run: sudo whoami
# Expected output: root
# Only proceed to 2.9 after confirming this works

# --- 2.9 Lock root account ---
passwd -l root
# Verify (locked hash starts with '!'):
grep root /etc/shadow | cut -d: -f2

# --- 2.10 PAM faillock hardening ---
# Check if faillock is already present (may have been applied by pam-auth-update):
grep -l 'pam_faillock' /etc/pam.d/common-auth /etc/pam.d/common-account 2>/dev/null
# If both files are listed — faillock is already wired in. Skip to faillock.conf below.
# If either file is missing from the output — add faillock manually:

if ! grep -q 'pam_faillock' /etc/pam.d/common-auth; then
  # Add preauth line before pam_unix.so:
  sed -i '/^auth.*pam_unix\.so/i auth\trequired\t\t\tpam_faillock.so preauth' \
    /etc/pam.d/common-auth
  # Add authfail line after pam_unix.so:
  sed -i '/^auth.*pam_unix\.so/a auth\t[default=die]\t\t\tpam_faillock.so authfail' \
    /etc/pam.d/common-auth
fi

if ! grep -q 'pam_faillock' /etc/pam.d/common-account; then
  echo 'account required    pam_faillock.so' >> /etc/pam.d/common-account
fi

# Configure faillock parameters:
sed -i 's/^#\s*deny\s*=.*/deny = 5/'                  /etc/security/faillock.conf
sed -i 's/^#\s*unlock_time\s*=.*/unlock_time = 900/'  /etc/security/faillock.conf
sed -i 's/^#\s*fail_interval\s*=.*/fail_interval = 900/' /etc/security/faillock.conf
# Verify:
grep -E 'deny|unlock_time|fail_interval' /etc/security/faillock.conf

# --- 2.11 Password quality policy ---
sed -i 's/^#\s*minlen\s*=.*/minlen = 12/'       /etc/security/pwquality.conf
sed -i 's/^#\s*minclass\s*=.*/minclass = 3/'    /etc/security/pwquality.conf
sed -i 's/^#\s*maxrepeat\s*=.*/maxrepeat = 3/'  /etc/security/pwquality.conf
sed -i 's/^#\s*usercheck\s*=.*/usercheck = 1/'  /etc/security/pwquality.conf
# Verify:
grep -vE '^#|^$' /etc/security/pwquality.conf
```

> ⚠️ **Critical order warning:** Always confirm `dan` can `sudo whoami` successfully in a separate session before running step 2.9. A locked root account with a broken sudo setup means a full OS reinstall.

> **PAM note:** If a future `pam-auth-update --force` run overwrites `/etc/pam.d/common-auth`, the manually added faillock lines may be replaced. After any `pam-auth-update` run, re-verify with `grep pam_faillock /etc/pam.d/common-auth`.

### [Back to Top](#section-order)

---

## Section 3 — Secure Shell (SSH)

> Goal: Make SSH access as narrow as possible. Only `dan` can connect, only via key authentication, on a non-standard port, with all unnecessary SSH features disabled and brute-force attempts automatically banned.

### Checklist

- [ ] **3.1** Generate an SSH key pair on your local machine
- [ ] **3.2** Deploy public key manually to `dan`'s authorized keys
- [ ] **3.3** Create SSH warning banner at `/etc/ssh/banner`
- [ ] **3.4** Harden `sshd_config`
- [ ] **3.5** Configure `fail2ban` for SSH
- [ ] **3.6** Open port 32022 in firewall
- [ ] **3.7** Restart `sshd` and verify new connection before closing current session

**Note on AppArmor and port 32022:** No AppArmor port registration is needed. Unlike SELinux, which enforces a strict type label (`ssh_port_t`) on every TCP port sshd is allowed to bind, AppArmor's sshd profile does not restrict which port the daemon binds to. Changing the SSH port to 32022 requires no AppArmor policy update — only the firewall and sshd_config changes below.

### Rationale

**3.1 & 3.2 — Key pair and manual deployment**  
Key authentication is non-negotiable. A private key is computationally infeasible to brute-force. `ed25519` is the correct algorithm choice: smaller, faster, and more secure than RSA 4096.

**3.3 — SSH warning banner**  
The banner is displayed to every client before authentication occurs. It establishes the legal basis for action against unauthorized users and makes clear the system is monitored. Permissions are set to `644` — it must be world-readable for `sshd` to serve it to unauthenticated connections.

**3.4 — `sshd_config` hardening**  
The OpenSSH default config is permissive for compatibility. Key changes: non-standard port, key-only auth, explicit user allowlist, tight algorithm selection, disabled unused forwarding features.

**3.5 — `fail2ban`**  
Even on a non-standard port with key-only auth, automated scanners will find port 32022 eventually. `fail2ban` watches `sshd` logs via systemd journal and bans IPs that exceed the failed attempt threshold via a `firewalld` rich rule. Configuration: 3 failed attempts → 1 hour ban. The `firewallcmd-rich-rules` banaction works identically on Debian when firewalld is the active firewall manager.

**3.6 — Open firewall port before restarting SSH**  
If you restart `sshd` with port 32022 configured but the firewall doesn't allow it yet, you lock yourself out. Firewall rule comes first, always.

**3.7 — Verify before closing session**  
Never close your existing SSH session until you've confirmed a new session connects successfully on port 32022. If anything is misconfigured, your old session is the recovery lifeline.

> **Optional escalation — Port Knocking:** Port knocking keeps SSH completely invisible to scanners until a specific sequence of ports is knocked in order. A daemon (`knockd`) then temporarily opens the SSH port for your IP only. This adds a lockout risk if the sequence or config is lost, so it is not prescribed in this guide but noted as an available escalation.

### Scriptable Commands

```bash
# --- 3.1 Generate key pair (run on your LOCAL machine, not the server) ---
ssh-keygen -t ed25519 -C "dan@your-hostname"
# Keys saved to: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# --- 3.2 Deploy public key manually ---
# Run on the SERVER as root:
mkdir -p /home/dan/.ssh
chmod 700 /home/dan/.ssh
nano /home/dan/.ssh/authorized_keys
# Paste your id_ed25519.pub content, save and exit
chmod 600 /home/dan/.ssh/authorized_keys
chown -R dan:dan /home/dan/.ssh

# --- 3.3 Create SSH warning banner ---
cat > /etc/ssh/banner << 'EOF'
WARNING: Unauthorized access to this system is prohibited.
All activity is monitored and logged.
EOF

chmod 644 /etc/ssh/banner
# Verify:
cat /etc/ssh/banner

# --- 3.4 Harden sshd_config ---
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

cat > /etc/ssh/sshd_config << 'EOF'
# --- Port & Protocol ---
Port 32022
AddressFamily any

# --- Banner ---
Banner /etc/ssh/banner

# --- Authentication ---
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30

# --- Access Control ---
AllowUsers dan

# --- Features (all disabled) ---
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PrintMotd no
UseDNS no

# --- Hardened Algorithms ---
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# --- Misc ---
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

# Verify config syntax before anything else:
sshd -t
# No output = no errors. Fix any errors before proceeding.

# No AppArmor port registration needed.
# AppArmor's sshd profile does not restrict port binding —
# port 32022 requires no policy update.

# --- 3.5 Configure fail2ban for SSH ---
cat > /etc/fail2ban/jail.d/sshd.local << 'EOF'
[sshd]
enabled   = true
port      = 32022
filter    = sshd
backend   = systemd
maxretry  = 3
bantime   = 3600
findtime  = 600
banaction = firewallcmd-rich-rules
EOF

systemctl enable --now fail2ban
# Verify:
fail2ban-client status sshd

# --- 3.6 Open port 32022 in firewall ---
systemctl enable --now firewalld
firewall-cmd --permanent --add-port=32022/tcp
firewall-cmd --reload
# Verify:
firewall-cmd --list-ports

# --- 3.7 Restart sshd ---
systemctl restart sshd
systemctl status sshd

# ⚠️  KEEP THIS SESSION OPEN
# Open a NEW terminal and test:
# ssh -p 32022 -i ~/.ssh/id_ed25519 dan@your-server-ip
# Banner should appear before the key prompt
# Only close this session after confirming the new connection works

# Post-restart: confirm AppArmor has no sshd denials:
journalctl -k --grep='apparmor.*DENIED.*sshd' --since boot
# Expected: no output
```

> ⚠️ **Critical order:** Firewall port (3.6) → restart `sshd` (3.7) → verify new session → close old session.

### [Back to Top](#section-order)

---

## Section 4 — Firewall

> Goal: Default-deny all inbound traffic. Only explicitly permitted ports are open. This section establishes the baseline firewall state — service-specific ports are added in their respective service guides layered on top of this one.

### Checklist

- [ ] **4.1** Verify `firewalld` is running and set to start on boot
- [ ] **4.2** Confirm the active zone is `public` and your interface is assigned to it
- [ ] **4.3** Remove default permitted services from the `public` zone
- [ ] **4.4** Confirm port 32022 is the only open port
- [ ] **4.5** Enable logging of denied traffic
- [ ] **4.6** Reload and verify final firewall state including IPv6 coverage

### Rationale

**4.1 — `firewalld` running state**  
`firewalld` was enabled in Section 3.5 as a prerequisite for SSH. This step confirms it's active and boot-enabled before further rule changes.

**4.2 — Active zone**  
`firewalld` uses zones to group interfaces with a trust level. The `public` zone assumes no trust of other hosts — correct for an internet-facing VPS.

**Default zone policies in `public`:**
- **Input:** `DROP` — inbound traffic is dropped unless explicitly permitted
- **Output:** `ACCEPT` — all outbound traffic is allowed by default
- **Forward:** `DROP` — packet forwarding between interfaces is dropped

**4.3 — Remove default services**  
`firewalld`'s default `public` zone on Debian may ship with `ssh` and `dhcpv6-client` permitted — VPS provider images vary. The `ssh` service covers port 22 which we have replaced with port 32022. Remove all defaults and keep only what this guide explicitly adds. Removal commands use `2>/dev/null || true` because the services may not be present in all provider images.

**4.4 — Port 32022 only**  
After removing defaults, port 32022/tcp from Section 3.6 should be the only open inbound path.

**4.5 — Denial logging**  
Enabling denial logging gives visibility into what's being blocked — useful during initial setup and periodic audits.

**4.6 — IPv6 coverage**  
`firewalld` manages both `iptables`/`ip6tables` (or `nftables`) automatically — the same zone rules apply to both stacks. On Debian 13, `firewalld` uses the `nftables` backend by default.

### Scriptable Commands

```bash
# --- 4.1 Verify firewalld is running ---
systemctl enable --now firewalld
systemctl status firewalld

# --- 4.2 Confirm active zone and interface assignment ---
firewall-cmd --get-active-zones
# Expected output example:
# public
#   interfaces: eth0

# If your interface is not in the public zone, assign it:
# firewall-cmd --permanent --zone=public --change-interface=eth0
# firewall-cmd --reload

# --- 4.3 Remove default permitted services ---
# Check what is currently permitted (varies by VPS provider):
firewall-cmd --permanent --zone=public --list-services

firewall-cmd --permanent --zone=public --remove-service=cockpit       2>/dev/null || true
firewall-cmd --permanent --zone=public --remove-service=dhcpv6-client 2>/dev/null || true
firewall-cmd --permanent --zone=public --remove-service=ssh           2>/dev/null || true
# ssh service covers port 22 — we use port 32022 exclusively (step 3.6)

# Verify nothing else is permitted:
firewall-cmd --permanent --zone=public --list-services
# Expected: (empty)

# --- 4.4 Confirm port 32022 is present (from Section 3.6) ---
firewall-cmd --permanent --zone=public --list-ports
# Expected: 32022/tcp

# If missing for any reason, re-add:
# firewall-cmd --permanent --zone=public --add-port=32022/tcp

# --- 4.5 Enable denial logging ---
firewall-cmd --set-log-denied=all

# --- 4.6 Reload and verify final state ---
firewall-cmd --reload

echo "=== Active Zones ==="
firewall-cmd --get-active-zones

echo "=== Show All Information ==="
firewall-cmd --zone=public --list-all

# Expected final state:
# Services  : (empty)
# Ports     : 32022/tcp
# Rich rules: (empty)

# Verify firewalld is managing IPv6:
firewall-cmd --info-zone=public | grep -i ipv
```

> **Note for service layering:** When adding services on top of this baseline, open only the exact ports needed:
> ```bash
> firewall-cmd --permanent --zone=public --add-port=<port>/tcp
> firewall-cmd --reload
> ```
> Never open port ranges or re-enable removed default services.

### [Back to Top](#section-order)

---

## Section 5 — Package Integrity

> Goal: Ensure every package on the system comes from a trusted, verified source and that no installed binary has been tampered with since installation.

### Checklist

- [ ] **5.1** Re-audit APT GPG verification across all active sources
- [ ] **5.2** Audit enabled repositories — disable any that are unnecessary or untrusted
- [ ] **5.3** Verify installed packages against shipped checksums via `debsums`
- [ ] **5.4** Remove orphaned packages

### Rationale

**5.1 — APT GPG re-audit**  
APT's GPG enforcement was confirmed before the initial upgrade in Section 1.4. This step performs a full re-audit now that all packages are installed and any provider-injected repos are visible. The goal is to confirm no source has been added with a `[trusted=yes]` override or a missing `Signed-By` field.

**5.2 — Repository audit**  
Every enabled repository is a potential source of packages installable onto your system. VPS providers typically inject their own agent or monitoring repos into `/etc/apt/sources.list.d/`. Review each one — disable anything you cannot justify.

**5.3 — `debsums`**  
`debsums -c` verifies installed files against the MD5 checksums shipped inside each `.deb` package (stored in `/var/lib/dpkg/info/*.md5sums`). This is Debian's equivalent of `rpm -Va`. Discrepancies on config files you modified during setup are expected. Discrepancies on binaries are not — investigate immediately.

Limitation: `debsums` relies on checksums packaged with the `.deb`. If both the binary and the `.deb` checksum were compromised together, `debsums` would not catch it. For a stronger integrity guarantee, AIDE (Section 11) provides ongoing filesystem monitoring from a baseline snapshot taken after hardening.

**5.4 — Orphaned packages**  
Packages installed as dependencies but no longer required by anything. `apt autoremove` removes them safely.

### Scriptable Commands

```bash
# --- 5.1 Full GPG re-audit across all sources ---
# Check for trusted=yes overrides:
grep -rE 'trusted=yes|allow-unauthenticated|AllowUnauthenticated' \
  /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
# Expected: no output

# Verify Debian archive signing key is present:
ls /etc/apt/trusted.gpg.d/
# Expected: debian-archive-*.gpg files

# Check global APT config for unauthenticated bypasses:
grep -r 'AllowUnauthenticated\|allow-unauthenticated' /etc/apt/apt.conf.d/ 2>/dev/null
# Expected: no output

# --- 5.2 Audit enabled repositories ---
# Traditional format (sources.list):
grep '^deb' /etc/apt/sources.list 2>/dev/null
# DEB822 format (debian.sources and any .sources files):
grep '^URIs:' /etc/apt/sources.list.d/*.sources 2>/dev/null
# Or use apt directly:
apt-cache policy

# Review each source — disable any you cannot justify:
# Comment out the line in the relevant file, or delete the .list file:
# nano /etc/apt/sources.list.d/<provider-repo>.list

# --- 5.3 Verify packages via debsums ---
debsums -c 2>/dev/null | tee /root/debsums-baseline.txt

# Output format: <filepath>  FAILED
# Expected: modified config files only (anything you changed during setup)
# Unexpected: modified binaries — investigate immediately

# Full audit including config files (more verbose):
# debsums -ca 2>/dev/null

# --- 5.4 Remove orphaned packages ---
apt autoremove -y
```

> **Baseline note:** Save `/root/debsums-baseline.txt` from step 5.3. After any major change or upgrade, re-run `debsums -c` and diff against this file to spot unexpected modifications. This complements AIDE (Section 11) which provides ongoing monitoring.

### [Back to Top](#section-order)

---

## Section 6 — Mandatory Access Control (MAC)

> Goal: Confirm AppArmor is loaded and that key service profiles are in enforce mode. Establish the correct approach to handling denials — targeted fixes, never disabling the module.

### Checklist

- [ ] **6.1** Verify AppArmor module is loaded and the service is active
- [ ] **6.2** Verify AppArmor is enabled at boot
- [ ] **6.3** Confirm the count and list of profiles in enforce mode
- [ ] **6.4** Review profiles in complain mode — enforce those covering running services
- [ ] **6.5** Audit AppArmor denials since boot and resolve any outstanding issues
- [ ] **6.6** Enforce any complain-mode profiles that should be enforced

### Rationale

**6.1 & 6.2 — AppArmor loaded and boot-enabled**  
Debian 13 ships with AppArmor compiled into the kernel and the `apparmor` service enabled by default. Unlike SELinux's single global mode (Enforcing/Permissive/Disabled), AppArmor is profile-based — each profile is independently set to enforce or complain mode. The `apparmor` service loads all profiles at boot; if the service is inactive, no profiles are loaded and no MAC enforcement is active.

**6.3 — Enforce mode profiles**  
On a fresh Debian 13 VPS, expect 20–40+ profiles loaded, most in enforce mode. These cover system services like `dhclient`, `cups`, `ntpd`, and on Debian 13, `sshd`. The exact count depends on installed packages — each package that ships an AppArmor profile adds to the list.

**6.4 — Complain mode profiles**  
Profiles in complain mode log denials without blocking. On a hardened baseline, every profile covering a service you actively run should be in enforce mode. Profiles for services not installed can safely remain in complain mode or be removed.

**6.5 — Denial audit**  
AppArmor denials appear in the kernel log with the tag `apparmor="DENIED"`. After all configuration changes in Sections 1–5, sweep for any denials — particularly around `sshd`, `fail2ban`, and `auditd`. A post-restart check for sshd denials was already done in Section 3.7; this step is a comprehensive sweep of everything since boot.

**6.6 — Enforce profiles**  
`aa-enforce` moves a profile from complain to enforce mode and reloads it. Local overrides for a profile go in `/etc/apparmor.d/local/<profile-name>` — this file is included by the main profile and survives package updates to the base profile.

> **The AppArmor cardinal rule:** When something breaks, the answer is never `systemctl stop apparmor` or `aa-complain /etc/apparmor.d/*`. Identify the specific denial with `journalctl -k | grep apparmor`, understand what is being blocked and why, then apply the narrowest possible fix — either a `aa-enforce` call, a custom rule in `/etc/apparmor.d/local/`, or a targeted `aa-complain` on just the affected profile while you diagnose. Disabling MAC system-wide to fix one misbehaving service trades your strongest mandatory access control layer for convenience.

### Scriptable Commands

```bash
# --- 6.1 Verify AppArmor module and service ---
# Module check (kernel parameter):
cat /sys/module/apparmor/parameters/enabled
# Expected: Y

# Service check:
systemctl is-active apparmor
systemctl status apparmor

# Full status (counts and profile names):
aa-status
# Expected output includes:
# "apparmor module is loaded."
# "N profiles are loaded."
# "X profiles are in enforce mode."

# --- 6.2 Verify AppArmor is enabled at boot ---
systemctl is-enabled apparmor
# Expected: enabled

# --- 6.3 List profiles in enforce mode ---
aa-status | grep 'profiles are in enforce mode'
# List enforce-mode profiles:
aa-status | awk '/in enforce mode/{found=1; next} found && /in complain mode/{exit} found{print}' \
  | grep '^\s'

# --- 6.4 Review complain-mode profiles ---
aa-status | grep 'profiles are in complain mode'
aa-status | awk '/in complain mode/{found=1; next} found && /processes have/{exit} found{print}' \
  | grep '^\s'
# For each profile in complain mode:
#   - Is the service running? Check with: systemctl is-active <service>
#   - If running and you care about it: enforce it (see step 6.6)
#   - If the service is not installed or not relevant: leave in complain

# --- 6.5 Audit AppArmor denials since boot ---
journalctl -k --grep='apparmor="DENIED"' --since boot
# No output = no denials — ideal state

# Alternative (dmesg):
dmesg | grep 'apparmor="DENIED"'

# Resolution hierarchy when denials exist:
#   1. Fix the service configuration if the denial is caused by misconfiguration
#   2. Add a local rule override: /etc/apparmor.d/local/<profile-name>
#   3. aa-complain /etc/apparmor.d/<profile> — temporary debug only
#   Never: systemctl stop apparmor

# --- 6.6 Enforce a specific profile ---
# Example — enforce the sshd profile if it is in complain mode:
# aa-enforce /etc/apparmor.d/usr.sbin.sshd

# Enforce all currently-loaded profiles at once (review complain list first):
# aa-enforce /etc/apparmor.d/*

# Reload a profile after making local overrides:
# apparmor_parser -r /etc/apparmor.d/<profile-file>

# Verify final state:
aa-status
# Confirm: 0 profiles in complain mode (or only profiles for non-installed services)
```

> **Local overrides:** To extend a profile without modifying the base file (which would be overwritten on package update), add rules to `/etc/apparmor.d/local/<profile-name>`. For example, to allow sshd a capability not in the base profile:
> ```bash
> echo 'capability net_admin,' >> /etc/apparmor.d/local/usr.sbin.sshd
> apparmor_parser -r /etc/apparmor.d/usr.sbin.sshd
> ```

### [Back to Top](#section-order)

---

## Section 7 — Logging & Audit

> Goal: Ensure all security-relevant events are captured, stored persistently, and rotated safely. Logs are your only forensic trail if something goes wrong.

### Checklist

- [ ] **7.1** Enable persistent storage in `journald`
- [ ] **7.2** Cap `journald` maximum disk usage
- [ ] **7.3** Configure `auditd` log retention
- [ ] **7.4** Deploy hardened `auditd` rules
- [ ] **7.5** Enable and start `auditd`
- [ ] **7.6** Verify log rotation is configured

### Rationale

**7.1 — `journald` persistent storage**  
By default on minimal installs, `journald` stores logs in memory only — all logs are lost on reboot. `Storage=persistent` writes logs to `/var/log/journal/`, surviving reboots.

**7.2 — `journald` disk cap**  
Persistent logs without a cap will grow unbounded and eventually fill the disk. `SystemMaxUse=500M` sets a hard ceiling.

**7.3 & 7.4 — `auditd` retention and rules**  
`auditd` hooks directly into the kernel and records system calls — giving visibility into file access, privilege escalation, user/group changes, and network configuration modifications. Configure retention (50MB per file, 10 files) and rules before starting so the first start picks up the complete configuration.

On Debian, `auditd` does **not** carry `RefuseManualStop=yes`. `auditd` can be restarted normally on Debian via `systemctl restart auditd`. The strict ordering constraint does not apply here, but pre-configuring before first start remains good practice.

**7.5 — Enable and start `auditd`**  
With retention config and rules in place, enable and start auditd together. It picks up everything from `auditd.conf` and `rules.d/` in one clean start.

**7.6 — Log rotation**  
Verify `logrotate` is handling system logs. Debian includes rotation configs for `auditd` and `rsyslog` by default.

### Scriptable Commands

```bash
# --- 7.1 Enable journald persistent storage ---
mkdir -p /var/log/journal
sed -i 's/^#Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
grep -q '^Storage=' /etc/systemd/journald.conf || \
  echo 'Storage=persistent' >> /etc/systemd/journald.conf

# --- 7.2 Cap journald disk usage ---
sed -i 's/^#SystemMaxUse=.*/SystemMaxUse=500M/' /etc/systemd/journald.conf
grep -q '^SystemMaxUse=' /etc/systemd/journald.conf || \
  echo 'SystemMaxUse=500M' >> /etc/systemd/journald.conf

systemctl restart systemd-journald
# Verify:
journalctl --disk-usage

# --- 7.3 Configure auditd log retention ---
sed -i 's/^max_log_file\s*=.*/max_log_file = 50/'                   /etc/audit/auditd.conf
sed -i 's/^num_logs\s*=.*/num_logs = 10/'                           /etc/audit/auditd.conf
sed -i 's/^max_log_file_action\s*=.*/max_log_file_action = ROTATE/' /etc/audit/auditd.conf

# Verify:
grep -E 'max_log_file|num_logs|max_log_file_action' /etc/audit/auditd.conf

# --- 7.4 Deploy auditd rules ---
mkdir -p /etc/audit/rules.d

cat > /etc/audit/rules.d/hardening.rules << 'EOF'
# --- Buffer size ---
-b 8192

# --- Failure mode: 1 = log, 2 = panic ---
-f 1

# --- Authentication & authorization ---
-w /etc/passwd          -p wa -k identity
-w /etc/shadow          -p wa -k identity
-w /etc/group           -p wa -k identity
-w /etc/gshadow         -p wa -k identity
-w /etc/sudoers         -p wa -k sudoers
-w /etc/sudoers.d/      -p wa -k sudoers

# --- SSH configuration ---
-w /etc/ssh/sshd_config -p wa -k sshd_config

# --- Privilege escalation ---
-w /bin/su              -p x  -k privilege_escalation
-w /usr/bin/sudo        -p x  -k privilege_escalation

# --- User and group management tools ---
-w /usr/sbin/useradd    -p x  -k user_mgmt
-w /usr/sbin/usermod    -p x  -k user_mgmt
-w /usr/sbin/userdel    -p x  -k user_mgmt
-w /usr/sbin/groupadd   -p x  -k user_mgmt
-w /usr/sbin/groupmod   -p x  -k user_mgmt
-w /usr/sbin/groupdel   -p x  -k user_mgmt

# --- Network configuration ---
-w /etc/hosts           -p wa -k network_config
-w /etc/resolv.conf     -p wa -k network_config

# --- Kernel module loading ---
-w /sbin/insmod         -p x  -k kernel_modules
-w /sbin/rmmod          -p x  -k kernel_modules
-w /sbin/modprobe       -p x  -k kernel_modules

# --- Login and session events ---
-w /var/log/lastlog     -p wa -k logins
-w /var/run/faillock/   -p wa -k logins

# --- Immutable flag: lock rules at runtime ---
# -e 2
# Uncomment only after rules are confirmed working.
# -e 2 makes rules immutable until next reboot.
# Leave commented during initial setup and first weeks of operation.
EOF

# --- 7.5 Enable and start auditd ---
# On Debian, auditd can be restarted normally — no RefuseManualStop restriction
systemctl enable --now auditd
systemctl status auditd

# Verify rules are active:
auditctl -l
# Expected: all rules from hardening.rules listed

# --- 7.6 Verify log rotation configs are present ---
ls /etc/logrotate.d/
# auditd and rsyslog entries should be present
cat /etc/logrotate.d/auditd
```

> **Querying audit logs:**
> ```bash
> ausearch -k identity              # passwd/shadow/group changes
> ausearch -k privilege_escalation  # all sudo/su usage
> ausearch -k sshd_config           # sshd_config modifications
> aureport --summary                # summarized report of all events
> ```

> **Updating rules on a running Debian system:** auditd on Debian accepts normal restarts:
> ```bash
> systemctl restart auditd
> auditctl -l  # verify rules reloaded
> ```

### [Back to Top](#section-order)

---

## Section 8 — Kernel Hardening 

> Goal: Harden the Linux kernel's network stack and core behavior against IP spoofing, SYN floods, redirect attacks, and information leakage via persistent `sysctl` parameters.

### Checklist

- [ ] **8.1** Disable IP forwarding
- [ ] **8.2** Disable ICMP redirect acceptance and sending
- [ ] **8.3** Disable source routing
- [ ] **8.4** Enable reverse path filtering
- [ ] **8.5** Enable SYN cookies
- [ ] **8.6** Enable martian packet logging
- [ ] **8.7** Harden kernel information exposure (`dmesg`, kernel pointers)
- [ ] **8.8** Harden ASLR and core dumps
- [ ] **8.9** Disable Magic SysRq key
- [ ] **8.10** Apply and persist all parameters

### Rationale

**8.1 — IP forwarding**  
IP forwarding allows the kernel to route packets between interfaces — necessary for routers, completely unnecessary for a VPS. Leaving it enabled means a compromised process could potentially route traffic through your server.

**8.2 — ICMP redirects**  
ICMP redirect messages instruct a host to update its routing table. A classic attack vector on public networks — a malicious actor can send crafted ICMP redirects to manipulate your routing table. Disable both accepting and sending redirects.

**8.3 — Source routing**  
IP source routing allows the sender to specify the route a packet takes. Almost exclusively used for spoofing and evasion today. No legitimate use case on a VPS.

**8.4 — Reverse path filtering**  
Validates that incoming packets arrive on the interface that would be used to reply to the source IP. Packets failing this check are likely spoofed and are dropped. `rp_filter = 1` enables strict mode — correct for a single-interface VPS.

**8.5 — SYN cookies**  
A SYN flood exhausts the server's connection table by sending large volumes of TCP SYN packets without completing the handshake. SYN cookies allow the kernel to handle SYN floods without allocating state for each half-open connection.

**8.6 — Martian packet logging**  
Martian packets have source addresses impossible on the public internet. Logging them gives visibility into spoofing attempts.

**8.7 — Kernel information exposure**  
`dmesg_restrict = 1` prevents unprivileged users from reading kernel ring buffer messages. `kptr_restrict = 2` hides kernel symbol addresses from all users in userspace — makes kernel exploit development significantly harder.

**8.8 — ASLR and core dumps**  
`randomize_va_space = 2` enables full Address Space Layout Randomization — the kernel randomizes the memory layout of every process, making memory corruption exploits dramatically harder. `fs.suid_dumpable = 0` disables core dumps for SUID processes — core dumps of privileged processes can expose sensitive memory contents including credentials.

**8.9 — Magic SysRq**  
On a VPS there is no physical keyboard, but the interface may be accessible via the provider's console. Disabling it removes an unnecessary privileged control path.

### Scriptable Commands

```bash
# --- 8.1 to 8.9 Write all security sysctl parameters ---
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# ============================================================
# Network hardening
# ============================================================

# 8.1 Disable IP forwarding
net.ipv4.ip_forward = 0

# 8.2 Disable ICMP redirect acceptance and sending
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# 8.3 Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# 8.4 Enable reverse path filtering (strict mode)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 8.5 Enable SYN cookies
net.ipv4.tcp_syncookies = 1

# 8.6 Martian packet logging
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# ============================================================
# Kernel hardening
# ============================================================

# 8.7 Restrict kernel information exposure
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2

# 8.8 Full ASLR + disable SUID core dumps
kernel.randomize_va_space = 2
fs.suid_dumpable = 0

# 8.9 Disable Magic SysRq
kernel.sysrq = 0
EOF

# --- 8.10 Apply all parameters ---
sysctl --system

# Verify a sample of applied values:
sysctl net.ipv4.ip_forward
sysctl net.ipv4.tcp_syncookies
sysctl kernel.randomize_va_space
sysctl kernel.dmesg_restrict
sysctl kernel.kptr_restrict
sysctl fs.suid_dumpable
sysctl kernel.sysrq
# Expected: all match values set above
```

> **On `99-hardening.conf`:** The `99-` prefix ensures this file is loaded last among all `sysctl.d` drop-ins, giving our values the highest precedence and overriding any defaults set by the VPS provider.

### [Back to Top](#section-order)

---

## Section 9 — Filesystem Hardening

> Goal: Restrict what can be executed and by whom on key filesystem paths, eliminate unnecessary privileged binaries, and lock critical configuration files against modification. All steps apply to an existing VPS without partition restructuring.

### Checklist

- [ ] **9.1** Harden `/tmp` mount options — `noexec`, `nosuid`, `nodev`
- [ ] **9.2** Harden `/dev/shm` mount options — `noexec`, `nosuid`, `nodev`
- [ ] **9.3** Harden `/proc` — restrict process visibility with `hidepid`
- [ ] **9.4** Audit and remove unnecessary SUID/SGID binaries
- [ ] **9.5** Sweep for world-writable files and directories
- [ ] **9.6** Apply `chattr +i` to immutable configuration files

### Rationale

**9.1 — `/tmp` hardening**  
`/tmp` is world-writable by design. Without mount restrictions, an attacker who gains write access can drop and execute a malicious binary. The `noexec`, `nosuid`, and `nodev` flags eliminate this. On Debian 13, `/tmp` is managed by systemd as a `tmpfs` mount — we override its options with a systemd drop-in.

**9.2 — `/dev/shm` hardening**  
`/dev/shm` is a shared memory filesystem used for inter-process communication. Like `/tmp`, it is world-writable and can be abused to stage and execute malicious code. This goes in `/etc/fstab` since it is not managed by a systemd unit.

**9.3 — `/proc` `hidepid`**  
By default every user can browse `/proc` and see all running processes — including command line arguments which frequently contain credentials or tokens. `hidepid=2` restricts `/proc/<pid>` entries so each user can only see their own processes.

**9.4 — SUID/SGID audit**  
Every unnecessary SUID binary is a potential privilege escalation path. Enumerate all of them, review the list, and remove the bit from anything that doesn't need it.

**9.5 — World-writable files**  
Outside of intentional shared paths like `/tmp` and `/dev/shm`, world-writable files are a misconfiguration that should be corrected.

**9.6 — `chattr +i` (immutable flag)**  
The immutable flag prevents a file from being modified, deleted, or renamed — even by root — until explicitly removed with `chattr -i`. Applied selectively to configuration files that should never change during normal operation.

Note: AppArmor profile files in `/etc/apparmor.d/` are **not** marked immutable — they need to be updateable by `apparmor_parser` during package updates and when applying local overrides.

### Scriptable Commands

```bash
# --- 9.1 Harden /tmp via systemd drop-in ---

# Check how /tmp is currently mounted:
findmnt /tmp
# If Type is tmpfs — use the systemd drop-in method below
# If Type is ext4/xfs — contact your provider

mkdir -p /etc/systemd/system/tmp.mount.d/
cat > /etc/systemd/system/tmp.mount.d/hardening.conf << 'EOF'
[Mount]
Options=mode=1777,strictatime,noexec,nosuid,nodev,size=50%
EOF

systemctl daemon-reload
systemctl restart tmp.mount

# Verify:
findmnt /tmp
# Expected options: noexec, nosuid, nodev

# --- 9.2 Harden /dev/shm via fstab ---
cp /etc/fstab /etc/fstab.bak2

# Check if /dev/shm already has an fstab entry:
grep shm /etc/fstab

# If no entry exists, add one:
echo 'tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0' >> /etc/fstab

# If an entry already exists, edit it manually:
# nano /etc/fstab

# Remount immediately without reboot:
mount -o remount,noexec,nosuid,nodev /dev/shm

# Verify:
findmnt /dev/shm

# --- 9.3 Harden /proc with hidepid ---
# Create a dedicated group for processes that need full /proc visibility:
groupadd -r procadmin
usermod -aG procadmin dan

# Add /proc hardening to fstab:
grep -q 'hidepid' /etc/fstab || \
  echo 'proc /proc proc defaults,hidepid=2,gid=procadmin 0 0' >> /etc/fstab

# Remount immediately:
mount -o remount,hidepid=2,gid=procadmin /proc

# Verify:
findmnt /proc

# --- 9.4 Audit SUID/SGID binaries ---
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f \
  2>/dev/null | tee /root/suid-sgid-baseline.txt

# Review the list carefully.
# Common legitimate SUID binaries on Debian (keep these):
# /usr/bin/passwd, /usr/bin/su, /usr/bin/sudo
# /usr/bin/newgrp, /usr/bin/chsh, /usr/bin/chage
# /usr/bin/ping, /usr/sbin/unix_chkpwd, /usr/bin/gpasswd

# Remove SUID/SGID bit from anything not explicitly needed:
# chmod u-s /path/to/binary   ← removes SUID
# chmod g-s /path/to/binary   ← removes SGID

# --- 9.5 World-writable file sweep ---
find / -xdev \
  -not \( -path '/tmp'     -prune \) \
  -not \( -path '/dev/shm' -prune \) \
  -not \( -path '/proc'    -prune \) \
  -not \( -path '/sys'     -prune \) \
  -perm -0002 -type f 2>/dev/null \
  | tee /root/world-writable-baseline.txt

# Review and correct each entry:
# chmod o-w /path/to/file

# Also check world-writable directories:
find / -xdev \
  -not \( -path '/tmp'     -prune \) \
  -not \( -path '/dev/shm' -prune \) \
  -not \( -path '/proc'    -prune \) \
  -not \( -path '/sys'     -prune \) \
  -perm -0002 -type d 2>/dev/null

# --- 9.6 Apply immutable flag to critical configs ---
chattr +i /etc/ssh/sshd_config
chattr +i /etc/ssh/banner
chattr +i /etc/sysctl.d/99-hardening.conf
chattr +i /etc/sysctl.d/99-performance.conf
chattr +i /etc/audit/rules.d/hardening.rules

# AppArmor profiles are NOT made immutable — they must remain
# updateable by apparmor_parser and package updates.

# Verify (the 'i' flag should be present):
lsattr /etc/ssh/sshd_config
lsattr /etc/sysctl.d/99-hardening.conf

# To temporarily modify any of these files later:
# chattr -i /path/to/file
# <make your changes>
# chattr +i /path/to/file
```

> **On `hidepid` and systemd:** Some systemd services read `/proc` entries of other processes during startup. If any service fails after applying `hidepid=2`, add it to the `procadmin` group: `usermod -aG procadmin <service-user>`.

### [Back to Top](#section-order)

---

## Section 10 — Performance Optimization

> Goal: Allocate swap space equal to physical RAM and tune kernel memory management parameters for a general-purpose VPS workload.

### Checklist

- [ ] **10.1** Detect physical RAM size in MiB
- [ ] **10.2** Create, format, and activate swapfile using `dd`
- [ ] **10.3** Persist swapfile in `/etc/fstab`
- [ ] **10.4** Tune `vm.swappiness`
- [ ] **10.5** Tune dirty page ratios
- [ ] **10.6** Tune inode/dentry cache pressure
- [ ] **10.7** Apply and persist performance `sysctl` parameters

### Rationale

**10.1 & 10.2 — Swapfile with `dd`**  
A swapfile provides an overflow buffer when physical RAM is exhausted. We use `dd` to write actual zeros to every block — unlike `fallocate` which only reserves space without writing data, leaving previously deleted data in those blocks. A swapfile created with `fallocate` could expose old data when the kernel writes memory pages into those blocks. `dd` guarantees the swapfile is clean. Permissions must be `600` — a world-readable swapfile exposes memory contents to any local user.

**10.3 — `/etc/fstab` persistence**  
`swapon` activates the swapfile for the current session only. Adding it to `/etc/fstab` ensures it is mounted automatically on every boot.

**10.4 — `vm.swappiness = 10`**  
Controls how aggressively the kernel moves memory pages to swap. The default is `60`. For a VPS where RAM is the primary performance resource, `10` tells the kernel to strongly prefer keeping data in RAM and only swap under genuine memory pressure.

**10.5 — Dirty page ratios**  
- `vm.dirty_background_ratio = 5` — at 5% of RAM dirty, background kernel threads start writing to disk quietly without blocking processes
- `vm.dirty_ratio = 15` — at 15% of RAM dirty, processes themselves are forced to flush

**10.6 — `vm.vfs_cache_pressure = 50`**  
Setting it to `50` makes the kernel favor keeping filesystem metadata in cache longer, benefiting workloads with frequent directory lookups and file access.

### Scriptable Commands

```bash
# --- 10.1 Detect RAM size in MiB ---
RAM_MiB=$(free -m | awk '/^Mem:/{print $2}')
echo "Detected RAM: ${RAM_MiB} MiB — swapfile will be ${RAM_MiB}M"

# --- 10.2 Create, format, and activate swapfile ---
dd if=/dev/zero of=/swapfile bs=1M count=${RAM_MiB} status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Verify:
swapon --show
free -h

# --- 10.3 Persist swapfile in fstab ---
cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Verify fstab entry:
grep swapfile /etc/fstab

# --- 10.4 to 10.6 Write performance sysctl parameters ---
cat > /etc/sysctl.d/99-performance.conf << 'EOF'
# ============================================================
# Memory management
# ============================================================

# 10.4 Reduce swap aggressiveness
vm.swappiness = 10

# 10.5 Dirty page flush thresholds
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15

# 10.6 Favor keeping inode/dentry cache
vm.vfs_cache_pressure = 50
EOF

# --- 10.7 Apply parameters ---
sysctl --system

# Verify applied values:
sysctl vm.swappiness
sysctl vm.dirty_background_ratio
sysctl vm.dirty_ratio
sysctl vm.vfs_cache_pressure
```

### [Back to Top](#section-order)

---

## Section 11 — Intrusion Detection System

> Goal: Establish a cryptographic baseline of the filesystem after hardening is complete. Any unexpected file modification, addition, or deletion is detectable on subsequent checks. Configured conservatively to minimize resource impact on a low-resource VPS.

### Resource Profile

| Operation | Duration | RAM Usage | CPU | Disk I/O |
|-----------|----------|-----------|-----|----------|
| `aide --init` | 2–5 min | ~80–120MB | 100% (1 core) | Heavy read |
| `aide --check` | 1–3 min | ~60–90MB | 100% (1 core) | Heavy read |
| Idle | — | 0 | 0 | None |

All scans are run with `nice -n 19 ionice -c 3` to minimize impact on live services. Schedule scans during your lowest-traffic window.

### Checklist

- [ ] **11.1** Configure AIDE to monitor only high-value paths
- [ ] **11.2** Build the initial AIDE database (`--init`)
- [ ] **11.3** Move database to active location
- [ ] **11.4** Run a baseline check to confirm database integrity
- [ ] **11.5** Create a resource-conscious check script
- [ ] **11.6** Schedule periodic checks via `cron`

### Rationale

**11.1 — Targeted monitoring scope**  
The default AIDE configuration monitors an extremely broad set of paths. On Debian 13, AIDE's config is at `/etc/aide/aide.conf`. We overwrite this with a lean, targeted configuration.

`database_attrs` suppresses GnuTLS GOST warnings. Setting `database_attrs` explicitly avoids these warnings when AIDE verifies its own database files.

**Attribute policies:**
- `NORMAL` — full integrity check including inode. Used for all standard paths.
- `BOOT` — identical to NORMAL but without inode checking (`i`). Used for `/boot` because the EFI partition uses FAT32, which generates synthetic inode numbers at mount time. These numbers change on every remount or reboot regardless of file content, generating guaranteed false positives. File content (checksums), permissions, and ownership are still fully monitored under `BOOT`.

The `xattrs` attribute (not `selinux`) is used for extended attribute monitoring — Debian uses AppArmor which does not store labels as SELinux xattrs. The `selinux` AIDE attribute is not applicable on a Debian AppArmor system.

**Monitored paths:** `/etc`, `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/boot`, `/root`, `/lib/modules`

**Excluded paths (high churn):** `/etc/mtab`, `/etc/resolv.conf`, `/var/log`, `/var/spool`, `/var/cache`, `/proc`, `/sys`, `/tmp`, `/dev/shm`, `/run`, volatile shell session files.

**11.2 — `aide --init` must run last**  
The database captures the current state as the trusted baseline. Run it only after all hardening steps are complete.

**11.3 — Database placement**  
AIDE writes the new database to `aide.db.new.gz`. It will not use it for checks until renamed to `aide.db.gz` — preventing AIDE from automatically trusting an unreviewed database.

### Scriptable Commands

```bash
# --- 11.1 Configure AIDE monitoring scope ---
# Debian AIDE config path: /etc/aide/aide.conf (not /etc/aide.conf)
cp /etc/aide/aide.conf /etc/aide/aide.conf.bak

cat > /etc/aide/aide.conf << 'EOF'
# AIDE configuration — Debian 13, resource-conscious, high-value paths only

# Database locations
database_in=file:/var/lib/aide/aide.db.gz
database_out=file:/var/lib/aide/aide.db.new.gz
database_new=file:/var/lib/aide/aide.db.new.gz
gzip_dbout=yes

# Database self-verification algorithms
# Explicitly listed to suppress stribog (GOST) warnings from GnuTLS
database_attrs=sha256+sha512+md5+sha1+rmd160

# Report output
report_url=file:/var/log/aide/aide.log
report_url=stdout

# ============================================================
# Attribute policies
# ============================================================

# Standard policy — full integrity check including inode
# xattrs used instead of selinux — Debian uses AppArmor (no SELinux xattrs)
NORMAL = sha256+sha512+ftype+p+i+n+u+g+s+acl+xattrs

# Boot policy — no inode checking
# /boot/efi uses FAT32 which generates synthetic inode numbers at mount time
BOOT = sha256+sha512+ftype+p+n+u+g+s+acl+xattrs

# ============================================================
# Monitored paths
# ============================================================
/boot                   BOOT
/bin                    NORMAL
/sbin                   NORMAL
/usr/bin                NORMAL
/usr/sbin               NORMAL
/lib/modules            NORMAL
/root                   NORMAL
/etc                    NORMAL

# ============================================================
# Explicit exclusions within monitored paths
# ============================================================
!/etc/mtab
!/etc/aide/aide.conf
!/etc/resolv.conf
!/var/log
!/var/spool
!/var/cache
!/proc
!/sys
!/tmp
!/dev/shm
!/run

# Volatile root home files — change on every session, not security-relevant
!/root/.lesshst
!/root/.zsh_history
!/root/.bash_history
!/root/.zsh_sessions
EOF

mkdir -p /var/log/aide
chmod 700 /var/log/aide

# --- 11.2 Build initial database ---
# ⚠️  Run ONLY after ALL hardening sections are complete
# ⚠️  If aide.conf is modified after this step, --init must be re-run
nice -n 19 ionice -c 3 aide --init
# Takes 2–5 minutes with pegged vCPU — expected behaviour

# --- 11.3 Activate the database ---
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# --- 11.4 Baseline integrity check ---
nice -n 19 ionice -c 3 aide --check
# Expected: AIDE found no differences between database and filesystem.

# --- 11.5 Create resource-conscious check script ---
cat > /usr/local/bin/aide-check.sh << 'EOF'
#!/bin/bash
# AIDE integrity check — resource-conscious wrapper

LOG=/var/log/aide/aide-check.log
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "========================================" >> "$LOG"
echo "AIDE check started: $TIMESTAMP"          >> "$LOG"
echo "========================================" >> "$LOG"

nice -n 19 ionice -c 3 aide --check >> "$LOG" 2>&1
EXIT_CODE=$?

echo "Exit code: $EXIT_CODE" >> "$LOG"
echo "AIDE check finished: $(date '+%Y-%m-%d %H:%M:%S')" >> "$LOG"

# AIDE exit codes are a bitmask:
# 0 = no differences
# 1 = added files
# 2 = removed files
# 4 = changed files
# 8+ = errors during execution
if [ $EXIT_CODE -eq 0 ]; then
  echo "RESULT: CLEAN — no changes detected"                             >> "$LOG"
elif [ $((EXIT_CODE & 8)) -ne 0 ] || [ $EXIT_CODE -ge 16 ]; then
  echo "RESULT: ERROR — AIDE encountered a problem (exit code: $EXIT_CODE)" >> "$LOG"
else
  echo "RESULT: CHANGES DETECTED — review log (exit code: $EXIT_CODE)"  >> "$LOG"
fi
EOF

chmod 700 /usr/local/bin/aide-check.sh
chown root:root /usr/local/bin/aide-check.sh

# --- 11.6 Schedule weekly check via cron ---
# Every Monday at 3:00 AM
echo '0 3 * * 1 root /usr/local/bin/aide-check.sh' \
  > /etc/cron.d/aide-check

chmod 644 /etc/cron.d/aide-check

# Verify:
cat /etc/cron.d/aide-check

# Run the check script once manually to verify it works:
bash /usr/local/bin/aide-check.sh

# Review result:
tail -5 /var/log/aide/aide-check.log
# Expected last line: RESULT: CLEAN — no changes detected
```

> **Updating the database after intentional changes:** Any time you deliberately modify a monitored file, rebuild the AIDE database afterward:
> ```bash
> nice -n 19 ionice -c 3 aide --update
> mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
> ```
> Habit: **change → verify → update database**.

> **Reviewing check logs:**
> ```bash
> tail -50 /var/log/aide/aide-check.log   # wrapper script log
> tail -50 /var/log/aide/aide.log          # AIDE native log
> ```

### [Back to Top](#section-order)

---

## Section 12 — Maintenance Hygiene

> Goal: Establish the operational habits that keep the hardened system hardened over time. A well-configured VPS degrades without a consistent maintenance rhythm.

### Checklist

- [ ] **12.1** Define snapshot discipline — before every significant change
- [ ] **12.2** Establish a weekly review routine
- [ ] **12.3** Establish a monthly review routine
- [ ] **12.4** Create a periodic hardening health-check script
- [ ] **12.5** Document your rollback procedure

### Rationale

**12.1 — Snapshot before changes**  
A snapshot taken before any significant change gives you a clean rollback point. Make it a non-negotiable reflex: **snapshot first, change second**.

Recommended triggers:
- Before any `apt upgrade` that includes kernel packages
- Before installing any new service
- Before modifying any hardening configuration
- Before running `aide --update`

**12.2 — Weekly review**  
Short, focused checks covering `fail2ban` ban activity, AIDE check results, and a brief auth log scan. Should take under 10 minutes.

**12.3 — Monthly review**  
Deeper checks covering open ports, running services, `auditd` summary, disk usage, and manual review of updates `unattended-upgrades` may have held back.

**12.4 — Health-check script**  
Runs the key verification commands from every hardening section and produces a concise status report. Particularly useful after a reboot or provider maintenance window.

**12.5 — Rollback procedure**  
Document the exact steps to restore from a snapshot for your specific provider. Write it down while calm, not during an incident.

### Scriptable Commands

```bash
# --- 12.1 Snapshot discipline ---
# Manual step at your VPS provider's control panel.
# Label snapshots with date and reason:
# e.g., "2025-05-01 pre-kernel-upgrade"

# --- 12.2 Weekly review commands ---
echo "=== fail2ban: banned IPs ==="
fail2ban-client status sshd

echo "=== AIDE: last check result ==="
tail -20 /var/log/aide/aide-check.log 2>/dev/null || \
  tail -20 /var/log/aide/aide.log

echo "=== Auth log: recent sudo and SSH events ==="
ausearch -k privilege_escalation -ts today 2>/dev/null | aureport --summary
journalctl _COMM=sshd --since "7 days ago" | grep -i 'failed\|accepted\|invalid'

echo "=== AppArmor: any new denials ==="
journalctl -k --grep='apparmor="DENIED"' --since "7 days ago" | head -20

# --- 12.3 Monthly review commands ---
echo "=== Open ports ==="
ss -tlnp

echo "=== Firewall state ==="
firewall-cmd --zone=public --list-all

echo "=== Running services ==="
systemctl list-units --type=service --state=running

echo "=== Auditd monthly summary ==="
aureport --summary \
  --start "$(date -d '1 month ago' '+%m/%d/%Y')" \
  --end   "$(date '+%m/%d/%Y')"

echo "=== Disk usage ==="
df -h
du -sh /var/log/audit/
du -sh /var/log/journal/

echo "=== Pending non-security updates ==="
apt-get --just-print upgrade 2>/dev/null | grep '^Inst' | head -20

echo "=== debsums drift check ==="
debsums -c 2>/dev/null | grep -v 'REPLACED'

echo "=== World-writable files drift check ==="
find / -xdev \
  -not \( -path '/tmp'     -prune \) \
  -not \( -path '/dev/shm' -prune \) \
  -not \( -path '/proc'    -prune \) \
  -not \( -path '/sys'     -prune \) \
  -perm -0002 -type f 2>/dev/null

echo "=== New SUID/SGID binaries since baseline ==="
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null \
  | diff /root/suid-sgid-baseline.txt - 2>/dev/null \
  | grep '^>'

# --- 12.4 Hardening health-check script ---
cat > /usr/local/bin/harden-check.sh << 'EOF'
#!/bin/bash
# Hardening posture health check — Debian 13, covers all 12 sections

PASS="\e[32mPASS\e[0m"
FAIL="\e[31mFAIL\e[0m"

check() {
  local label="$1"
  local result="$2"
  local expected="$3"
  if echo "$result" | grep -qE "$expected"; then
    echo -e "[$PASS] $label"
  else
    echo -e "[$FAIL] $label"
    echo "       Got: $result"
  fi
}

echo "================================================"
echo " Hardening Health Check — $(date '+%Y-%m-%d %H:%M:%S')"
echo "================================================"

# Section 1 — OS Baseline
check "timesyncd active"          "$(systemctl is-active systemd-timesyncd)"              "active"
check "unattended-upgrades on"    "$(systemctl is-enabled unattended-upgrades)"            "enabled"

# Section 2 — User & Auth
check "root account locked"       "$(passwd -S root | awk '{print $2}')"                  "L"
check "dan in sudo group"         "$(groups dan)"                                          "sudo"

# Section 3 — SSH
check "sshd running"              "$(systemctl is-active sshd)"                            "active"
check "sshd on port 32022"        "$(ss -tlnp | grep sshd)"                               "32022"
check "PasswordAuth off"          "$(sshd -T | grep passwordauthentication)"               "no"
check "fail2ban running"          "$(systemctl is-active fail2ban)"                        "active"

# Section 4 — Firewall
check "firewalld running"         "$(systemctl is-active firewalld)"                       "active"
check "port 32022 open"           "$(firewall-cmd --zone=public --list-ports)"             "32022"

# Section 6 — AppArmor
check "AppArmor module loaded"    "$(cat /sys/module/apparmor/parameters/enabled 2>/dev/null)" "Y"
check "AppArmor service active"   "$(systemctl is-active apparmor)"                        "active"

# AppArmor sshd denial check (0 = pass):
SSH_DENIALS=$(journalctl -k --grep='apparmor="DENIED".*sshd' --since boot 2>/dev/null | wc -l)
if [ "$SSH_DENIALS" -eq 0 ]; then
  echo -e "[$PASS] No AppArmor sshd denials"
else
  echo -e "[$FAIL] AppArmor sshd denials found: $SSH_DENIALS"
fi

# Section 7 — Audit & Logging
check "auditd running"            "$(systemctl is-active auditd)"                          "active"
check "auditd rules loaded"       "$(auditctl -l | wc -l)"                                 "[1-9]"
check "journald persistent"       "$(journalctl --disk-usage | grep 'Archived')"           "M|G"

# Section 8 — Kernel hardening
check "ip_forward disabled"       "$(sysctl -n net.ipv4.ip_forward)"                      "0"
check "SYN cookies enabled"       "$(sysctl -n net.ipv4.tcp_syncookies)"                   "1"
check "ASLR enabled"              "$(sysctl -n kernel.randomize_va_space)"                 "2"
check "dmesg restricted"          "$(sysctl -n kernel.dmesg_restrict)"                     "1"

# Section 9 — Filesystem Hardening
check "/tmp noexec"               "$(findmnt /tmp     | grep noexec)"                      "noexec"
check "/dev/shm noexec"           "$(findmnt /dev/shm | grep noexec)"                      "noexec"

# Section 10 — Performance
check "swap active"               "$(swapon --show | grep swapfile)"                       "swapfile"
check "swappiness at 10"          "$(sysctl -n vm.swappiness)"                             "10"

# Section 11 — AIDE
check "AIDE database exists"      "$(ls /var/lib/aide/aide.db.gz 2>/dev/null)"             "aide.db.gz"
check "AIDE cron scheduled"       "$(cat /etc/cron.d/aide-check 2>/dev/null)"              "aide-check.sh"

echo "================================================"
echo " Check complete"
echo "================================================"
EOF

chmod 700 /usr/local/bin/harden-check.sh
chown root:root /usr/local/bin/harden-check.sh

# Run immediately to confirm baseline passes:
bash /usr/local/bin/harden-check.sh

# --- 12.5 Rollback procedure template ---
cat > /root/ROLLBACK.md << 'EOF'
# VPS Rollback Procedure

## Provider snapshot restore
1. Log in to provider control panel
2. Navigate to: [your VPS > Snapshots]
3. Select snapshot labeled with date and reason
4. Click Restore — confirm when prompted
5. Wait for restore to complete (typically 2–5 minutes)
6. SSH in to verify: ssh -p 32022 -i ~/.ssh/id_ed25519 dan@<ip>

## After restore — verify hardening posture
bash /usr/local/bin/harden-check.sh

## If SSH is unreachable after restore
1. Use provider console (web-based VNC/serial)
2. Log in as dan
3. Check sshd:    systemctl status sshd
4. Check firewall: firewall-cmd --list-ports
5. Check AppArmor: systemctl status apparmor && aa-status

## Emergency: if locked out completely
1. Provider console → log in as dan
2. sudo systemctl restart sshd
3. If sshd fails: sudo aa-complain /etc/apparmor.d/usr.sbin.sshd → restart → diagnose → re-enforce
EOF

chmod 600 /root/ROLLBACK.md
echo "Rollback procedure written to /root/ROLLBACK.md"

# ROLLBACK.md was created after AIDE --init — update the database:
nice -n 19 ionice -c 3 aide --update
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
echo "AIDE database updated to include ROLLBACK.md"
```

### [Back to Top](#section-order)

---

## Quick Reference — Section Execution Order

Run sections in this exact order. Do not skip ahead.

```
1  → OS Baseline           (updates, essentials, unattended-upgrades)
2  → User & Auth           (create dan, lock root)
3  → SSH Hardening         (keys, sshd_config, fail2ban — no AppArmor port step)
4  → Firewall              (firewalld baseline rules)
5  → Package Integrity     (APT GPG re-audit, debsums)
6  → AppArmor              (confirm loaded, audit profiles and denials)
7  → Audit & Logging       (auditd rules, journald persistent)
8  → Kernel Hardening      (sysctl security)
9  → Filesystem Hardening  (tmp/shm/proc, SUID sweep, chattr)
10 → Performance           (swapfile, sysctl performance)
11 → AIDE                  (init database LAST — after all changes)
12 → Maintenance Hygiene   (scripts, cron, rollback doc)
```

> ⚠️ **AIDE `--init` must be the final hardening action.** The database captures the trusted baseline state — any change made after `--init` will appear as a false alert on the next check.

### [Back to Top](#section-order)
