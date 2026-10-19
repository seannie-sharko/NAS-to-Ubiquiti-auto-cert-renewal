# Automated Synology Let's Encrypt Renewal Through an EdgeRouter X

A reproducible, privacy-safe guide to temporarily expose **TCP/80** for Synology DSM HTTP-01 certificate validation, automatically close it afterward, and independently verify cleanup.

## Contents

1. [Architecture and design](#architecture-and-design)
2. [Requirements and example values](#requirements-and-example-values)
3. [What was done](#what-was-done)
4. [Router prerequisites and SSH access](#router-prerequisites-and-ssh-access)
5. [Router opening and closing scripts](#router-opening-and-closing-scripts)
6. [Watchdog and EdgeOS scheduler](#watchdog-and-edgeos-scheduler)
7. [Synology renewal wrapper](#synology-renewal-wrapper)
8. [Testing safely](#testing-safely)
9. [DSM scheduling and notifications](#dsm-scheduling-and-notifications)
10. [Troubleshooting](#troubleshooting)
11. [Maintenance and security](#maintenance-and-security)

## Architecture and design

```text
Let's Encrypt HTTP-01 (Internet)
           |
           | inbound TCP/80, only during renewal
           v
    EdgeRouter X (EdgeOS 3.0.1)
    WAN: public IPv4 (not included here)
           |
           | temporary port-forward rule 9
           v
    Synology DSM NAS: 192.0.2.20:80  (EXAMPLE ONLY)
           |
           +-- /usr/syno/sbin/syno-letsencrypt renew-all

NAS (root) -- SSH key + restricted noninteractive sudo --> EdgeRouter
       open_80.sh --> arm 30-minute deadline, create live DNAT/FW/NAT
       close_80.sh --> remove rule + stale kernel entries, save closed config
       renew-synology-certs.sh --> timeout, inspect, verify, always cleanup

EdgeRouter cron (every minute)
       le-port80-watchdog.sh --> close orphaned/expired TCP/80 rules
```

**Timing:** The router deadline is 1,800 seconds (30 minutes); the NAS `renew-all` timeout is 1,380 seconds (23 minutes). This allows time for network operations and cleanup. The watchdog detects expiry on its next one-minute run, not necessarily to-the-second, and depends on the router, cron and scripts being functional.

**Rule persistence:** The open operation intentionally does **not** save its configuration. The close operation saves the **closed** state to `/config/config.boot`, so rebooting the router should not re-enable temporary forwarding.

**Scope of exposed port:** The forwarding is to the NAS's port 80, not DSM administration port 5000/5001. Keep DSM and other services protected independently. HTTP-01 requires valid public DNS records and reachable TCP/80 from the Internet; an internal/hairpin test alone is not sufficient.

## Requirements and example values

| Parameter | Example placeholder | Notes |
|---|---|---|
| Router firmware | EdgeOS 3.0.1 | Observed working firmware |
| Router LAN address | `192.0.2.1` | **Replace**; RFC 5737 documentation-only address |
| Synology LAN address | `192.0.2.20` | **Replace**; use static IP/DHCP reservation |
| Router SSH user | `routeradmin` | **Replace** with a non-root account |
| Primary domain | `nas.example.org` | **Replace** with a domain you control |
| Other service names | `home.example.org`, `nas-alt.example.org`, `media.example.org`, `cloud.example.org`, `sync.example.org` | **Replace** with actual certificate SAN hostnames |
| Forward rule | `9` | Reserve this ID; do not overwrite an unrelated rule |
| WAN interface | `eth0` | Verify on your device |
| LAN/hairpin interfaces | `eth1`, `switch0` | Verify actual interfaces/ipsets |
| Router SSH key (NAS) | `/root/.ssh/edgerouter_renew` | Never publish private key |
| Router scripts | `/config/scripts/` | Survive EdgeOS firmware changes depending on configuration |
| NAS wrapper | `/usr/local/bin/renew-synology-certs.sh` | Run as root via DSM scheduler |

> **IMPORTANT:** `192.0.2.0/24` is not a deployable home LAN example. Replace `192.0.2.1` and `192.0.2.20` throughout the files **before running anything**. Likewise, edit the monitored hostname list.

Tools used: Bash, SSH, OpenSSL 1.1.1w, `/bin/timeout`, EdgeOS's `/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper` and `/bin/cli-shell-api`, `/sbin/iptables`, `/usr/bin/flock`, cron, Synology `/usr/syno/sbin/syno-letsencrypt`. Check their paths before adopting this document on another firmware/DSM version.

## What was done

1. Confirmed router WAN/port-forward rules, NAS reachability, public DNS, and the HTTP-01 validation path.
2. Created a dedicated Ed25519 SSH identity on the NAS, installed the public key on the router, and verified passwordless SSH and `sudo -n` for router scripts.
3. Implemented temporary EdgeOS **rule 9** mapping external TCP/80 to NAS TCP/80.
4. Discovered that `#!/bin/vbash` plus `configure; set; commit; save` could update EdgeOS *configuration* without creating the actual running iptables DNAT/FW rules. Switched to `vyatta-cfg-cmd-wrapper` and verified kernel rules with `iptables -C`.
5. Removed stale DNAT, FW and hairpin SNAT rules that sometimes survived removal of the EdgeOS configuration rule.
6. Added a **deadline file**, a **one-minute watchdog**, and a **shared `flock` lock** across the open/close/watchdog scripts.
7. Fixed EdgeOS task-scheduler validation by making the watchdog executable/readable to the invoking account (`root:root`, `755`).
8. Fixed a Bash descriptor-detection bug in the lock handoff: `/proc/self/fd/9` and string `readlink` comparison were unreliable. The final close script uses `/proc/$$/fd/9 -ef "$LOCK"` to identify the inherited lock file.
9. Added a Synology wrapper with a preflight, timeout, output/error checks, certificate hostname and validity checks, EXIT/interrupt cleanup, and independent router-state verification.
10. Confirmed six public service certificates matched their expected hostnames and had more than 30 days remaining; observed a successful full renewal wrapper with verified closing and exit code 0.

## Router prerequisites and SSH access

### Examine router configuration without changing it

```bash
show configuration commands | grep 'port-forward'
show configuration commands | grep 'task-scheduler'
sudo iptables -t nat -S UBNT_PFOR_DNAT_RULES
sudo iptables -S UBNT_PFOR_FW_RULES
sudo iptables -t nat -S UBNT_PFOR_SNAT_RULES
command -v flock
```

The original setup used EdgeOS `port-forward auto-firewall`, WAN `eth0`, and hairpin NAT on `eth1` and `switch0`. Inspect your topology and ensure forwarding rule **9** is not assigned to another service before starting.

### Install key-based access

On the **NAS** (root), if you do not already have a dedicated key:

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
ssh-keygen -t ed25519 -f /root/.ssh/edgerouter_renew -C 'synology-acme-router' -N ''
chmod 600 /root/.ssh/edgerouter_renew
cat /root/.ssh/edgerouter_renew.pub
```

Install **only the public key** on your EdgeRouter user's SSH authorized keys using EdgeOS's supported mechanism (for example `loadkey` / CLI settings on your firmware). Do not put the private key on the router. Confirm the router host key interactively once, then require `StrictHostKeyChecking=yes` in automation.

From the NAS, replace host and user:

```bash
ssh -i /root/.ssh/edgerouter_renew \
  -o IdentitiesOnly=yes -o BatchMode=yes \
  routeradmin@192.0.2.1 'sudo -n /config/scripts/close_80.sh'
```

**Privilege security:** The tested router account had noninteractive sudo available. For a new deployment, prefer a carefully scoped `sudoers` rule for the specific root-owned scripts, ideally with separate read-only verification privileges, rather than blanket passwordless sudo. Test remote sudo noninteractively. Never place secrets in the scripts or logs.

## Router opening and closing scripts

All router scripts are root-owned. **Replace `NAS_IP` and related addresses before executing.**

### `/config/scripts/open_80.sh`

```bash
#!/bin/bash
set -u
umask 077

CFG="/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper"
API="/bin/cli-shell-api"
IPT="/sbin/iptables"
LOCK="/var/run/le-port80.lock"
DEADLINE="/config/scripts/.le-port80-deadline"
NAS_IP="192.0.2.20"                    # REPLACE
TIMEOUT=1800

fail() { echo "ERROR: $*" >&2; exit 1; }
[ "$(id -u)" -eq 0 ] || fail "Root required"
exec 9>"$LOCK" || fail "Cannot open lock"
/usr/bin/flock -w 20 9 || fail "Router lock busy"
for tool in "$CFG" "$API" "$IPT"; do
    [ -x "$tool" ] || fail "Missing executable: $tool"
done

"$API" existsActive port-forward rule 9 && fail "Rule 9 already active"
[ ! -e "$DEADLINE" ] || fail "A deadline already exists"

# Check entire dedicated port-forward chains for stale TCP/80 entries.
for spec in \
    'nat UBNT_PFOR_DNAT_RULES' \
    'filter UBNT_PFOR_FW_RULES' \
    'nat UBNT_PFOR_SNAT_RULES'; do
    set -- $spec
    rules=$("$IPT" -t "$1" -S "$2") || fail "Cannot read chain $2"
    if printf '%s\n' "$rules" | grep -Eq -- '--dport 80( |$)'; then
        fail "Stale TCP/80 rule in $2; run close_80.sh first"
    fi
done

# ARM BEFORE OPENING. Atomic rename avoids partial deadline reads.
expiry=$(( $(date +%s) + TIMEOUT ))
tmp="${DEADLINE}.$$"
printf '%s\n' "$expiry" > "$tmp" || fail "Cannot write deadline"
mv "$tmp" "$DEADLINE" || fail "Cannot arm deadline"
echo "Watchdog armed for 30 minutes; opening TCP/80..."

"$CFG" begin || fail "EdgeOS begin failed"
if ! "$CFG" set port-forward rule 9 description OPEN_80 ||
   ! "$CFG" set port-forward rule 9 forward-to address "$NAS_IP" ||
   ! "$CFG" set port-forward rule 9 forward-to port 80 ||
   ! "$CFG" set port-forward rule 9 original-port 80 ||
   ! "$CFG" set port-forward rule 9 protocol tcp ||
   ! "$CFG" commit; then
    "$CFG" end
    fail "Configuration failed; manually run close_80.sh"
fi
"$CFG" end || fail "EdgeOS end failed"
"$API" existsActive port-forward rule 9 || fail "Rule 9 missing after commit"
"$IPT" -t nat -C UBNT_PFOR_DNAT_RULES -p tcp -m tcp --dport 80 \
    -j DNAT --to-destination "${NAS_IP}:80" || fail "DNAT missing"
"$IPT" -t filter -C UBNT_PFOR_FW_RULES -d "${NAS_IP}/32" \
    -p tcp -m tcp --dport 80 -j ACCEPT || fail "Firewall rule missing"

echo "SUCCESS: TCP/80 forwarding active; deadline: $expiry"
# Intentionally DO NOT SAVE an open forwarding rule.
```

### `/config/scripts/close_80.sh`

This is the **corrected inherited-descriptor implementation** from the final successful watchdog test. The `-ef` file-identity comparison matters: replacing it with literal `readlink` path comparison recreated a 20-second lock timeout and deadlock in the original testing.

```bash
#!/bin/bash
set -u
umask 077

CFG="/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper"
API="/bin/cli-shell-api"
IPT="/sbin/iptables"
FLOCK="/usr/bin/flock"
LOCK="/var/run/le-port80.lock"
DEADLINE="/config/scripts/.le-port80-deadline"
NAS_IP="192.0.2.20"  # REPLACE

fail() { echo "ERROR: $*" >&2; exit 1; }
[ "$(id -u)" -eq 0 ] || fail "Root required"
for tool in "$CFG" "$API" "$IPT" "$FLOCK"; do
    [ -x "$tool" ] || fail "Missing executable: $tool"
done

# The watchdog calls this script while holding FD 9.
# Compare file ID (not textual path); /var/run may resolve to /run.
if [ -e "/proc/$$/fd/9" ] && [ "/proc/$$/fd/9" -ef "$LOCK" ]; then
    echo "Using inherited router lock."
else
    echo "Acquiring router lock..."
    exec 9>"$LOCK" || fail "Cannot open router lock"
fi
"$FLOCK" -w 20 9 || fail "Cannot acquire router lock"

echo "Removing port-forward rule 9..."
if "$API" existsActive port-forward rule 9; then
    "$CFG" begin || fail "EdgeOS begin failed"
    if ! "$CFG" delete port-forward rule 9 || ! "$CFG" commit; then
        "$CFG" end
        fail "Cannot delete/commit rule 9"
    fi
    "$CFG" end || fail "EdgeOS end failed"
fi

echo "Cleaning up stale NAT and firewall rules..."
remove_rule() {
    local table="$1"
    shift
    while "$IPT" -t "$table" -C "$@" 2>/dev/null; do
        "$IPT" -t "$table" -D "$@" || return 1
    done
}

remove_rule nat UBNT_PFOR_DNAT_RULES \
    -p tcp -m tcp --dport 80 \
    -j DNAT --to-destination "${NAS_IP}:80" || fail "DNAT cleanup failed"

remove_rule filter UBNT_PFOR_FW_RULES \
    -d "${NAS_IP}/32" -p tcp -m tcp --dport 80 \
    -j ACCEPT || fail "Firewall cleanup failed"

remove_rule nat UBNT_PFOR_SNAT_RULES \
    -d "${NAS_IP}/32" -o eth1 -p tcp \
    -m set --match-set NETv4_eth1 src -m tcp --dport 80 \
    -j MASQUERADE || fail "eth1 SNAT cleanup failed"

remove_rule nat UBNT_PFOR_SNAT_RULES \
    -d "${NAS_IP}/32" -o switch0 -p tcp \
    -m set --match-set NETv4_switch0 src -m tcp --dport 80 \
    -j MASQUERADE || fail "switch0 SNAT cleanup failed"

echo "Verifying TCP/80 cleanup..."
if "$API" existsActive port-forward rule 9; then
    fail "Rule 9 still active"
fi
for spec in \
    'nat UBNT_PFOR_DNAT_RULES' \
    'filter UBNT_PFOR_FW_RULES' \
    'nat UBNT_PFOR_SNAT_RULES'; do
    set -- $spec
    rules=$("$IPT" -t "$1" -S "$2") || fail "Cannot inspect chain $2"
    if printf '%s\n' "$rules" | grep -Eq -- '--dport 80( |$)'; then
        fail "TCP/80 remains in $2"
    fi
done

echo "Saving closed EdgeOS configuration..."
"$CFG" save || fail "Could not save closed configuration"
rm -f "$DEADLINE" || fail "Could not remove deadline"
echo "SUCCESS: TCP 80 forwarding closed and verified."
```

> **Safety note:** `-ef` verifies the identity of the inherited descriptor's file, not independently whether the descriptor is locked. The follow-up `flock 9` is what enforces locking. Keep the scripts root-owned and writable only by root. Avoid untrusted privileged callers passing arbitrary descriptors.

## Watchdog and EdgeOS scheduler

### `/config/scripts/le-port80-watchdog.sh`

```bash
#!/bin/bash
set -u

API="/bin/cli-shell-api"
IPT="/sbin/iptables"
CLOSE="/config/scripts/close_80.sh"
LOCK="/var/run/le-port80.lock"
DEADLINE="/config/scripts/.le-port80-deadline"
LOG="/config/scripts/le-port80-watchdog.log"
NAS_IP="192.0.2.20"  # REPLACE

log() { echo "$(date '+%F %T') $*" >> "$LOG"; }
[ "$(id -u)" -eq 0 ] || exit 1

exec 9>"$LOCK" || exit 1
# Skip a tick if open/close owns the lock; next tick retries.
/usr/bin/flock -n 9 || exit 0

port80_active() {
    "$API" existsActive port-forward rule 9 && return 0
    "$IPT" -t nat -C UBNT_PFOR_DNAT_RULES -p tcp -m tcp \
        --dport 80 -j DNAT --to-destination "${NAS_IP}:80" 2>/dev/null && return 0
    "$IPT" -t filter -C UBNT_PFOR_FW_RULES -d "${NAS_IP}/32" \
        -p tcp -m tcp --dport 80 -j ACCEPT 2>/dev/null && return 0
    "$IPT" -t nat -C UBNT_PFOR_SNAT_RULES -d "${NAS_IP}/32" -o eth1 \
        -p tcp -m set --match-set NETv4_eth1 src -m tcp --dport 80 \
        -j MASQUERADE 2>/dev/null && return 0
    "$IPT" -t nat -C UBNT_PFOR_SNAT_RULES -d "${NAS_IP}/32" -o switch0 \
        -p tcp -m set --match-set NETv4_switch0 src -m tcp --dport 80 \
        -j MASQUERADE 2>/dev/null && return 0
    return 1
}

close_and_verify() {
    # Child inherits the locked FD 9; close_80.sh recognizes its file identity.
    if ! "$CLOSE" >> "$LOG" 2>&1; then
        log "CRITICAL: Closing script failed."
        return 1
    fi
    if port80_active; then
        log "CRITICAL: TCP/80 still active after cleanup."
        return 1
    fi
    log "SUCCESS: TCP 80 forwarding closed."
}

now=$(date +%s)
if [ ! -f "$DEADLINE" ]; then
    if port80_active; then
        log "WARNING: Forwarding active without deadline."
        close_and_verify || exit 1
    fi
    exit 0
fi

expiry=$(cat "$DEADLINE")
case "$expiry" in
    ''|*[!0-9]*)
        log "WARNING: Invalid deadline file; attempting cleanup."
        expiry=0
        ;;
esac

[ "$now" -lt "$expiry" ] && exit 0
log "Port 80 watchdog deadline expired."
close_and_verify || exit 1
[ ! -e "$DEADLINE" ] || { log "CRITICAL: Deadline still exists."; exit 1; }
log "Watchdog cleanup completed."
```

### Permissions and syntax validation

```bash
sudo chown root:root /config/scripts/{open_80.sh,close_80.sh,le-port80-watchdog.sh}
sudo chmod 750 /config/scripts/open_80.sh /config/scripts/close_80.sh
sudo chmod 755 /config/scripts/le-port80-watchdog.sh
sudo bash -n /config/scripts/open_80.sh
sudo bash -n /config/scripts/close_80.sh
sudo bash -n /config/scripts/le-port80-watchdog.sh
```

The watchdog must be visible as executable to EdgeOS configuration validation. `700 root:root` caused the actual scheduler setup to fail with `File ... does not exist or is not executable`, followed by a `must define executable` commit error. `755 root:root` worked.

### EdgeOS recurring task

In the router CLI:

```text
configure
set system task-scheduler task le-port80-watchdog executable path /config/scripts/le-port80-watchdog.sh
set system task-scheduler task le-port80-watchdog interval 1m
commit
save
exit
```

Verify:

```bash
show configuration commands | grep 'le-port80-watchdog'
sudo grep 'le-port80-watchdog' /etc/cron.d/vyatta-crontab
sudo systemctl is-active cron
```

Observed generated cron format:

```text
*/1 * * * * root /config/scripts/le-port80-watchdog.sh
```

## Synology renewal wrapper

Save as `/usr/local/bin/renew-synology-certs.sh` on the **NAS** and edit the router account/address and all monitored hostnames.

```bash
#!/bin/bash
set -u
umask 077

ROUTER="routeradmin@192.0.2.1"     # REPLACE
SSH_KEY="/root/.ssh/edgerouter_renew"
LOG="/var/log/synology-cert-renew.log"
LOCKDIR="/tmp/synology-cert-renew.lock"
ARCHIVE="/usr/syno/etc/certificate/_archive"
RENEW_TIMEOUT=1380
MIN_VALIDITY=2592000   # 30 days
OPEN_ATTEMPTED=0
RUN_OUTPUT=""

SSH_OPTS=(
    -i "$SSH_KEY"
    -o IdentitiesOnly=yes
    -o BatchMode=yes
    -o StrictHostKeyChecking=yes
    -o ConnectTimeout=10
    -o ServerAliveInterval=10
    -o ServerAliveCountMax=3
)

log() { echo "$(date '+%F %T') $*" | tee -a "$LOG"; }
fail() { log "ERROR: $*"; exit 1; }
router() {
    ssh "${SSH_OPTS[@]}" "$ROUTER" \
        "sudo -n /config/scripts/$1" >> "$LOG" 2>&1
}

router_closed() {
    ssh "${SSH_OPTS[@]}" "$ROUTER" \
        'sudo -n /bin/sh -s' >> "$LOG" 2>&1 <<'REMOTE'
API="/bin/cli-shell-api"
IPT="/sbin/iptables"
if "$API" existsActive port-forward rule 9; then
    echo "ERROR: EdgeOS forwarding rule 9 remains"
    exit 1
fi
for spec in 'nat UBNT_PFOR_DNAT_RULES' 'filter UBNT_PFOR_FW_RULES' 'nat UBNT_PFOR_SNAT_RULES'; do
    set -- $spec
    rules=$("$IPT" -t "$1" -S "$2") || exit 1
    if printf '%s\n' "$rules" | grep -Eq -- '--dport 80( |$)'; then
        echo "ERROR: Port 80 rule remains in $2"
        exit 1
    fi
done
echo 'Router verification PASSED: TCP 80 closed.'
REMOTE
}

cleanup() {
    local result=$?
    trap - EXIT INT TERM HUP
    if [ "$OPEN_ATTEMPTED" -eq 1 ]; then
        log "Closing EdgeRouter TCP 80..."
        if router close_80.sh; then
            log "Router closing script completed."
        else
            log "CRITICAL: Router closing script failed."
            result=1
        fi
        if router_closed; then
            log "Router closed-state verification PASSED."
        else
            log "CRITICAL: Router closed-state verification FAILED."
            result=1
        fi
    fi
    [ -z "$RUN_OUTPUT" ] || rm -f "$RUN_OUTPUT"
    rmdir "$LOCKDIR" 2>/dev/null || true
    log "Finished. Exit status: $result"
    exit "$result"
}

check_certificate() {
    local hostname="$1" cert found=0
    while IFS= read -r -d '' cert; do
        # On tested DSM OpenSSL, checkhost output is matched explicitly.
        if ! /bin/openssl x509 -in "$cert" -noout \
            -checkhost "$hostname" 2>/dev/null |
            grep -Fxq "Hostname $hostname does match certificate"; then
            continue
        fi
        found=1
        if /bin/openssl x509 -in "$cert" -noout \
            -checkend "$MIN_VALIDITY" >/dev/null 2>&1; then
            log "CERT PASS: $hostname"
            /bin/openssl x509 -in "$cert" -noout -enddate >> "$LOG" 2>&1
            return 0
        fi
    done < <(find "$ARCHIVE" -type f -name cert.pem -print0)
    if [ "$found" -eq 0 ]; then
        log "CERT FAIL: No matching archive certificate for $hostname"
    else
        log "CERT FAIL: $hostname expires within 30 days"
    fi
    return 1
}

verify_certificates() {
    local hostname failed=0
    log 'Verifying certificate validity...'
    for hostname in \
        home.example.org \
        nas.example.org \
        nas-alt.example.org \
        media.example.org \
        cloud.example.org \
        sync.example.org; do
        check_certificate "$hostname" || failed=1
    done
    return "$failed"
}

[ "$(id -u)" -eq 0 ] || fail 'Root required'
[ -x /bin/timeout ] || fail 'timeout unavailable'
[ -x /bin/openssl ] || fail 'OpenSSL unavailable'
[ -x /usr/syno/sbin/syno-letsencrypt ] || fail 'Synology ACME tool unavailable'
[ -f "$SSH_KEY" ] || fail 'SSH key missing'
[ -d "$ARCHIVE" ] || fail 'Certificate archive missing'
mkdir "$LOCKDIR" 2>/dev/null || fail 'Renewal already running or stale lock'
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
trap 'exit 129' HUP
RUN_OUTPUT=$(mktemp /tmp/syno-letsencrypt.XXXXXX) || fail 'Cannot create temporary output'

log "Starting Synology Let's Encrypt renewal."
OPEN_ATTEMPTED=1
log 'Opening router TCP/80...'
router open_80.sh || fail 'Router failed to open TCP/80'
sleep 5
log 'Running syno-letsencrypt renew-all...'
RENEW_RC=0
/bin/timeout "$RENEW_TIMEOUT" /usr/syno/sbin/syno-letsencrypt renew-all \
    > "$RUN_OUTPUT" 2>&1 || RENEW_RC=$?
cat "$RUN_OUTPUT" >> "$LOG"
[ "$RENEW_RC" -ne 124 ] || fail 'Renewal exceeded timeout'
[ "$RENEW_RC" -eq 0 ] || fail "Renewal exited $RENEW_RC"

# Advisory error signature check: not an exhaustive ACME parser.
if grep -Eiq \
    'Timeout during connect|challenge.*(fail|error)|authorization.*(fail|invalid)|certificate.*(fail|error)|"error"[[:space:]]*:[[:space:]]*[1-9]' \
    "$RUN_OUTPUT"; then
    fail 'Possible ACME error in renewal output'
fi

log 'Renewal command completed without detected errors.'
verify_certificates || fail 'One or more certificates failed verification'
log 'All monitored certificates passed verification.'
log 'Renewal processing completed successfully.'
```

Install with:

```bash
chown root:root /usr/local/bin/renew-synology-certs.sh
chmod 700 /usr/local/bin/renew-synology-certs.sh
bash -n /usr/local/bin/renew-synology-certs.sh && echo 'Syntax PASSED'
```

**Important limitations:**

- `renew-all` can exit `0` even after individual validation failures; the error signature filter is only a heuristic, and the 30-day validity check does **not** prove the most recent issuance attempt succeeded. Check actual output and issuance/expiry dates when investigating.
- The monitored-host list does **not** include Synology-managed self-signed certificates or vendor-managed QuickConnect credentials. Add any additional *public Let's Encrypt* hostnames that should be monitored.
- The `timeout` utility's default signal behavior may not terminate every descendant process. Investigate hung process trees before trusting a strict maximum runtime.
- The NAS `mkdir` lock under `/tmp` can remain after an unclean reboot/crash; inspect processes before removing a stale lock. Use `flock` if extending this to more robust scheduling.
- The 30-minute timer starts when the router **opens**, not when the NAS command starts; SSH/network overhead consumes some of the seven-minute margin.

## Testing safely

### A. Read-only baseline (router)

```bash
show configuration commands | grep 'port-forward rule 9'
sudo iptables-save -t nat | grep -E -- '--dport 80|--dports 80'
sudo iptables -S UBNT_PFOR_FW_RULES | grep -- '--dport 80'
sudo test ! -e /config/scripts/.le-port80-deadline && echo 'Deadline absent'
```

Normally the first three commands produce **no output**. If another legitimate service intentionally uses TCP/80, stop and redesign instead of deleting unrelated rules.

### B. Basic open/close smoke test (router)

```bash
sudo /config/scripts/open_80.sh
sudo cat /config/scripts/.le-port80-deadline
sudo iptables -t nat -S UBNT_PFOR_DNAT_RULES | grep -- '--dport 80'
sudo iptables -S UBNT_PFOR_FW_RULES | grep -- '--dport 80'
sudo /config/scripts/close_80.sh
```

Run the close command immediately after checking; do not leave the test port unnecessarily exposed. Verify that the port-forward config and iptables TCP/80 entries are absent and the deadline was removed.

### C. Expired-deadline test, without opening the port

Do this only after confirming port 80 is already closed:

```bash
sudo sh -c 'echo $(( $(date +%s) - 60 )) > /config/scripts/.le-port80-deadline'
if sudo /config/scripts/le-port80-watchdog.sh; then
    echo 'Watchdog lock test PASSED'
else
    echo 'Watchdog lock test FAILED'
fi
sudo tail -n 25 /config/scripts/le-port80-watchdog.log
sudo test ! -e /config/scripts/.le-port80-deadline && echo 'Deadline cleanup PASSED'
```

For a real **cron** test, create another expired deadline while closed and let the next minute tick execute automatically (do **not** invoke the watchdog manually). Expect log entries indicating expiry, inherited lock, cleanup, and success.

### D. Read-only NAS-to-router verification

From NAS root, edit the example login/address:

```bash
ssh -i /root/.ssh/edgerouter_renew \
  -o IdentitiesOnly=yes -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  routeradmin@192.0.2.1 'sudo -n /bin/sh -s' <<'REMOTE'
API=/bin/cli-shell-api
IPT=/sbin/iptables
if "$API" existsActive port-forward rule 9; then
    echo 'FAIL: forwarding config active'; exit 1
fi
for spec in 'nat UBNT_PFOR_DNAT_RULES' 'filter UBNT_PFOR_FW_RULES' 'nat UBNT_PFOR_SNAT_RULES'; do
    set -- $spec
    rules=$("$IPT" -t "$1" -S "$2") || exit 1
    if printf '%s\n' "$rules" | grep -Eq -- '--dport 80( |$)'; then
        echo "FAIL: TCP/80 entry in $2"; exit 1
    fi
done
echo 'PASS: EdgeRouter TCP 80 forwarding is closed.'
REMOTE
```

The original deployment produced that final `PASS` message over SSH.

### E. Inspect archived certificates without renewing

```bash
find /usr/syno/etc/certificate/_archive -type f -name cert.pem \
  -exec /bin/openssl x509 -in {} -noout -subject -enddate \;
```

Test individual hostname matching and 30-day validity on the appropriate certificate file:

```bash
/bin/openssl x509 -in /path/to/cert.pem -noout -checkhost nas.example.org
/bin/openssl x509 -in /path/to/cert.pem -noout -checkend 2592000
```

On the observed DSM version, successful hostname output included `Hostname nas.example.org does match certificate`; return semantics may differ across OpenSSL variants. Validate this behavior when upgrading.

**Avoid repeated forced renewals solely to test automation.** Let's Encrypt may impose issuance/rate limits.

## DSM scheduling and notifications

In **DSM → Control Panel → Task Scheduler**:

1. Create a **Scheduled Task → User-defined script**.
2. Task name: `Let's Encrypt Certificate Renewal`.
3. Run as: `root`.
4. Suggested schedule: **weekly, Sunday, 03:00 local time**, provided this does not overlap other DSM renewal/backup jobs.
5. User-defined script:

   ```bash
   /bin/bash /usr/local/bin/renew-synology-certs.sh
   ```

6. Configure DSM's outgoing mail service and choose notifications for abnormal termination (exact UI labels depend on DSM version).
7. Check the log after the first actual scheduled execution:

   ```bash
   tail -n 80 /var/log/synology-cert-renew.log
   ```

Successful end-to-end log tail observed in the original deployment:

```text
All monitored certificates passed verification.
Renewal processing completed successfully.
Closing EdgeRouter TCP 80...
SUCCESS: TCP 80 forwarding closed and verified.
Router closing script completed.
Router verification PASSED: TCP 80 closed.
Router closed-state verification PASSED.
Finished. Exit status: 0
```

This demonstrates a successful **wrapper run** and cleanup; it does not mean all six certificates were reissued in that run.

## Troubleshooting

| Symptom | Likely explanation | What to check / resolution |
|---|---|---|
| ACME `Timeout during connect` | Internet TCP/80 is blocked, WAN NAT missing, upstream firewall, or DNS wrong | Check public A/AAAA records and WAN address; test from cellular/outside LAN; sniff NAS TCP/80 with `tcpdump`; compare iptables running chains to EdgeOS configuration |
| HTTP on port 80 redirects to another host | nginx/DSM reverse proxy or port configuration changes validation path | Check destination `/.well-known/acme-challenge/`; ensure the expected host/path is reachable externally during validation; avoid unexpected cross-host redirects |
| EdgeOS shows rule 9, but `iptables` has no TCP/80 | EdgeOS configuration commit did not program kernel forwarding | Use `vyatta-cfg-cmd-wrapper begin/set/commit/end`; verify DNAT/FW via `iptables -C` rather than relying only on `show configuration commands` |
| Rule 9 removed but port 80 NAT persists | Orphaned DNAT/FW/SNAT rules in Ubiquiti chains | Run the verified `close_80.sh`, which removes exact-match entries for the intended NAS and verifies no matching TCP/80 rules remain |
| Scheduler says file does not exist or is not executable | Watchdog was root-only mode `700`; EdgeOS CLI validates with less privilege | Set watchdog `root:root`, `chmod 755`; `bash -n`; rerun scheduler `commit` and `save` |
| Scheduler `must define executable` | Previous executable-path `set` failed, leaving incomplete config | Delete/discard uncommitted task; repair permissions; configure executable path again; commit/save |
| Cron task exists but no cleanup yet | Cron minute boundary not reached; watchdog lock busy; script error | Inspect `/etc/cron.d/vyatta-crontab`, `systemctl is-active cron`, `journalctl -u cron`, watchdog log; confirm deadline actually expired |
| `date: invalid date '+2 minutes'` | EdgeRouter `date` does not accept GNU relative-date syntax | Use arithmetic: `$(date +%s)` plus/minus seconds in `$(( ... ))` |
| Empty deadline after failed `date` | Shell redirected output before command failed | If port is closed, remove/recreate deadline. Invalid deadlines trigger fail-closed cleanup in watchdog |
| Watchdog says `Cannot acquire router lock` after 20 seconds | Close script attempted a *second* lock while watchdog already held FD 9 | Use `"/proc/$$/fd/9" -ef "$LOCK"` to detect inherited file identity and run `flock` on FD 9; avoid textual `readlink` comparison and `/proc/self/fd/9` |
| Watchdog reports success but no fresh log line | With no deadline and no open rules, watchdog exits silently | Add an expired test deadline while closed and check the next cron run |
| `renew-all` returns `0` despite individual failures | Synology CLI status can be insufficient | Check actual command output for ACME errors and archive certs' hostname and expiry; investigate any certificate approaching the 30-day threshold |
| Script refuses with stale NAS lock | Previous process crashed before `rmdir` | Confirm **no renewal process is running** (`ps`, process tree) before removing `/tmp/synology-cert-renew.lock`; do not delete lock during active issuance |
| Remote SSH fails in scheduled job | Incorrect key permissions, host key, noninteractive sudo, user, or network access | Test SSH as NAS root with `BatchMode=yes`, `StrictHostKeyChecking=yes`; verify authorized public key and narrow sudo permissions |
| Router cleanup succeeds, but remote verification fails | Independent audit detected residual rule or SSH/read error | Check SSH logs, EdgeOS rule 9, all three port-forward chains; run close script and investigate before scheduling again |
| Shell `echo $?` appears garbled | Interactive terminal paste or input issue, not necessarily script failure | Test via `cat -v`; instead run `if command; then echo PASS; else echo FAIL; fi`; don't infer success from a malformed `printf` invocation |

### Helpful router commands

```bash
# Config and scheduler
show configuration commands | grep 'port-forward rule 9'
show configuration commands | grep 'le-port80-watchdog'
sudo grep 'le-port80-watchdog' /etc/cron.d/vyatta-crontab
sudo systemctl is-active cron

# Live iptables state
sudo iptables -t nat -S UBNT_PFOR_DNAT_RULES
sudo iptables -S UBNT_PFOR_FW_RULES
sudo iptables -t nat -S UBNT_PFOR_SNAT_RULES

# Watchdog history and deadline
sudo tail -n 60 /config/scripts/le-port80-watchdog.log
sudo cat /config/scripts/.le-port80-deadline

# Closed-state recovery (safe only for dedicated rule 9 and intended NAT rules)
sudo /config/scripts/close_80.sh
```

### Helpful NAS commands

```bash
# Script checks
bash -n /usr/local/bin/renew-synology-certs.sh
ls -l /root/.ssh/edgerouter_renew
command -v timeout
openssl version

# Renewal log and certificate inventory
tail -n 100 /var/log/synology-cert-renew.log
find /usr/syno/etc/certificate/_archive -type f -name cert.pem \
  -exec /bin/openssl x509 -in {} -noout -subject -dates \;

# Inspect possible overlapping processes
ps aux | grep -E 'syno-letsencrypt|renew-synology-certs' | grep -v grep
```

### External HTTP-01 troubleshooting

The deployment originally encountered connection timeout errors and a 301 redirect to a different hostname. After NAT fixes, a WAN-side cellular HTTP request reached NAS nginx and returned HTTP 200. When testing:

- Verify public DNS, IP-family reachability (IPv4 **and** IPv6 if AAAA exists), inbound ISP/network filtering, and the NAS's listener on port 80.
- Use an actual **outside** client, not just the LAN: hairpin NAT may hide external failures.
- Capture packets on the NAS only during controlled tests, for example `sudo tcpdump -ni any 'tcp port 80'` if available; stop captures afterward.
- Inspect DSM reverse proxy/http-to-https redirects and the specific challenge path. A generic HTTP 200 at `/` confirms reachability, **not** necessarily correct ACME challenge handling.
- For failures, read `/var/log/synology-cert-renew.log` and use the router's watchdog log to distinguish ACME, SSH, forwarding and cleanup stages.

## Maintenance and security

- **Never publish** private SSH keys, real public IP addresses, customer LAN diagrams, or raw domain/certificate inventories when sharing troubleshooting logs. This README uses documentation-only IPs and `example.org` hostnames.
- Keep scripts root-owned; watchdog `755` is needed for the observed EdgeOS config validator. Other router scripts may be `750`.
- Confirm no other service uses forwarding rule 9 or matching TCP/80 iptables entries. The cleanup scripts intentionally operate on dedicated Ubiquiti port-forward chains.
- Back up scripts, router config, Synology certificate export/configuration, and key material securely. Do not include a private key in plain-text backups shared with others.
- Whenever router or DSM firmware changes, rerun syntax, permission, `iptables` chain-name, `flock`, and scheduler checks. DSM can change internal certificate storage paths and CLI semantics.
- Verify the **first scheduled** DSM run and periodically inspect failed renewal notifications. Monitor real certificate expiration independently if possible; a shell exit code alone is not enough.
- If a renewal exceeds 23 minutes, inspect the CA/network response rather than simply increasing the timeout; keep enough margin before the router's 30-minute fail-safe.
- Consider a future improvement to minimize the SSH `sudo` attack surface and replace the NAS `mkdir` lock with a crash-safe process/file lock.

## Observed final outcome

The anonymized deployment concluded with a successful wrapper log showing six certificate validity checks passing, router close succeeding, an independent router iptables/config audit confirming port 80 closed, and final exit code `0`. The scheduled EdgeOS watchdog independently handled expired deadlines and successfully called the corrected close script with an inherited lock. This is a documented working reference, **not** a promise that all firmware revisions or network topologies will behave identically.
