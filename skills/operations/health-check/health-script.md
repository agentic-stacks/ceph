# Automated Health Script

Save this as `/usr/local/bin/ceph-health-check.sh` on any admin node. Run it from cron or a CI pipeline.

```bash
#!/usr/bin/env bash
# ceph-health-check.sh — Ceph cluster health check script
# Exit codes: 0 = OK, 1 = WARN, 2 = ERR, 3 = script error
# Usage: ./ceph-health-check.sh [--json] [--quiet]
set -euo pipefail

SCRIPT_NAME="$(basename "$0")"
JSON_OUTPUT=false
QUIET=false
EXIT_CODE=0

for arg in "$@"; do
  case "$arg" in
    --json)   JSON_OUTPUT=true ;;
    --quiet)  QUIET=true ;;
  esac
done

log()  { $QUIET || echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
warn() { echo "[WARN]  $*" >&2; EXIT_CODE=$((EXIT_CODE > 1 ? EXIT_CODE : 1)); }
err()  { echo "[ERROR] $*" >&2; EXIT_CODE=2; }

# --- Verify ceph is reachable ---
if ! timeout 10 ceph status &>/dev/null; then
  echo "[FATAL] Cannot reach Ceph cluster (timeout or auth failure)" >&2
  exit 3
fi

# --- Collect cluster status ---
HEALTH_STATUS=$(ceph health 2>/dev/null)
HEALTH_LEVEL=$(echo "$HEALTH_STATUS" | awk '{print $1}')

log "=== Ceph Health Check ==="
log "Status: $HEALTH_STATUS"

# --- Check overall health ---
case "$HEALTH_LEVEL" in
  HEALTH_OK)
    log "Cluster is healthy"
    ;;
  HEALTH_WARN)
    warn "Cluster in HEALTH_WARN: $HEALTH_STATUS"
    log "Detail:"
    ceph health detail 2>/dev/null | while IFS= read -r line; do log "  $line"; done
    ;;
  HEALTH_ERR)
    err "Cluster in HEALTH_ERR: $HEALTH_STATUS"
    log "Detail:"
    ceph health detail 2>/dev/null | while IFS= read -r line; do log "  $line"; done
    ;;
  *)
    echo "[FATAL] Unexpected health output: $HEALTH_STATUS" >&2
    exit 3
    ;;
esac

# --- OSD status ---
OSD_STAT=$(ceph osd stat 2>/dev/null)
log ""
log "=== OSD Status ==="
log "$OSD_STAT"

OSD_UP=$(echo "$OSD_STAT" | grep -oP '\d+(?= up)')
OSD_IN=$(echo "$OSD_STAT" | grep -oP '\d+(?= in)')
OSD_TOTAL=$(echo "$OSD_STAT" | grep -oP '^\d+')

if [[ "$OSD_UP" -lt "$OSD_TOTAL" ]]; then
  OSDS_DOWN=$(( OSD_TOTAL - OSD_UP ))
  warn "$OSDS_DOWN OSD(s) are down (up=${OSD_UP}, total=${OSD_TOTAL})"
fi

if [[ "$OSD_IN" -lt "$OSD_TOTAL" ]]; then
  OSDS_OUT=$(( OSD_TOTAL - OSD_IN ))
  log "Note: ${OSDS_OUT} OSD(s) are out of cluster (may be intentional)"
fi

# --- PG status ---
log ""
log "=== PG Status ==="
PG_STAT=$(ceph pg stat 2>/dev/null)
log "$PG_STAT"

if echo "$PG_STAT" | grep -qvE "active\+clean"; then
  UNCLEAN=$(ceph pg dump_stuck inactive 2>/dev/null | grep -c "^[0-9]" || true)
  STALE=$(ceph pg dump_stuck stale 2>/dev/null | grep -c "^[0-9]" || true)
  DEGRADED=$(ceph pg dump_stuck degraded 2>/dev/null | grep -c "^[0-9]" || true)
  [[ "$UNCLEAN" -gt 0 ]] && warn "${UNCLEAN} inactive PGs"
  [[ "$STALE" -gt 0 ]]   && warn "${STALE} stale PGs"
  [[ "$DEGRADED" -gt 0 ]] && warn "${DEGRADED} stuck degraded PGs"
fi

# --- Capacity ---
log ""
log "=== Capacity ==="
USAGE_LINE=$(ceph df 2>/dev/null | grep "^TOTAL")
log "$USAGE_LINE"

RAW_USED=$(ceph df 2>/dev/null | awk '/^TOTAL/{print $NF}' | tr -d '%')
if [[ -n "$RAW_USED" ]]; then
  if (( $(echo "$RAW_USED > 90" | bc -l) )); then
    err "Cluster raw utilization critical: ${RAW_USED}% used"
  elif (( $(echo "$RAW_USED > 80" | bc -l) )); then
    warn "Cluster raw utilization high: ${RAW_USED}% used"
  else
    log "Raw utilization: ${RAW_USED}% (OK)"
  fi
fi

# --- Crash reports ---
log ""
log "=== Crash Reports ==="
CRASH_COUNT=$(ceph crash ls 2>/dev/null | grep -c "NEW" || true)
if [[ "$CRASH_COUNT" -gt 0 ]]; then
  warn "${CRASH_COUNT} unacknowledged crash report(s)"
  ceph crash ls 2>/dev/null | while IFS= read -r line; do log "  $line"; done
  log "  Run: ceph crash info <id>  to investigate"
  log "  Run: ceph crash archive-all  to acknowledge after investigation"
else
  log "No unacknowledged crash reports"
fi

# --- Service status (requires cephadm orchestrator) ---
if ceph orch status &>/dev/null 2>&1; then
  log ""
  log "=== Service Status ==="
  STOPPED=$(ceph orch ps 2>/dev/null | grep -c "stopped\|error\|crash" || true)
  if [[ "$STOPPED" -gt 0 ]]; then
    warn "${STOPPED} daemon(s) stopped or in error state"
    ceph orch ps 2>/dev/null | grep -E "stopped|error|crash" | \
      while IFS= read -r line; do log "  $line"; done
  else
    log "All daemons running"
  fi
fi

# --- Summary ---
log ""
log "=== Summary ==="
case "$EXIT_CODE" in
  0) log "Result: OK — cluster is healthy" ;;
  1) log "Result: WARNING — attention required" ;;
  2) log "Result: ERROR — immediate action required" ;;
esac

if $JSON_OUTPUT; then
  python3 -c "
import subprocess, json, sys
s = json.loads(subprocess.check_output(['ceph', '-s', '-f', 'json']))
print(json.dumps({'health': s.get('health', {}), 'osdmap': s.get('osdmap', {})}, indent=2))
"
fi

exit "$EXIT_CODE"
```

Make it executable and test:

```bash
chmod +x /usr/local/bin/ceph-health-check.sh
ceph-health-check.sh                  # standard output
ceph-health-check.sh --json           # include JSON cluster summary
ceph-health-check.sh --quiet 2>&1 | grep -E "WARN|ERROR"  # silent mode
echo $?                                # 0=OK 1=WARN 2=ERR 3=unreachable
```

Schedule via cron (run as root or ceph admin user):

```
# /etc/cron.d/ceph-health
*/5 * * * * root /usr/local/bin/ceph-health-check.sh --quiet >> /var/log/ceph/health-check.log 2>&1
```

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/health-checks.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/monitoring.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/operations/monitoring-osd-pg.rst
-->
