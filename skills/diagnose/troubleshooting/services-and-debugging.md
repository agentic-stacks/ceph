# Service Issues and Debugging

Troubleshooting guides for MDS, RGW service issues, log locations, and collecting debug information.

**When to use this guide:** An MDS or RGW service is down or misbehaving, you need to find logs, or you need to collect debug information for a bug report.

---

## MDS Not Active

**Symptom:** CephFS mounts fail, or `ceph status` shows no active MDS:
```
mds: <none>
```
or:
```
mds: 1/1 daemons up, 0 standby
# followed by client mount failures
```

**Step 1 — Check the MDS status:**
```bash
ceph mds stat
# Reports: up:active={X} or no active MDS

ceph fs status
# Shows filesystem name, active rank, standby daemons, pools
```

**Step 2 — If no MDS daemon is running:**
```bash
# Cephadm:
ceph orch ps | grep mds
ceph orch daemon restart mds.<id>

# Systemd:
systemctl status ceph-mds@<id>.service
journalctl -u ceph-mds@<id> --since "30 minutes ago" | tail -50
```

**Step 3 — If MDS is running but stuck in `replay` or `reconnect` state:**
```bash
ceph mds stat
# State: replay or reconnect

# MDS is replaying the journal after a crash. It will proceed automatically.
# If stuck more than 10 minutes:
ceph tell mds.<id> flush journal
```

**Step 4 — If MDS is slow or causing client timeouts:**
```bash
# Check MDS cache usage:
ceph daemon mds.<id> status | grep -E "cache|mds_mem"

# If cache is thrashing, increase the MDS cache size:
ceph config set mds mds_cache_memory_limit 4294967296  # 4 GB
```

---

## RGW Not Responding

**Symptom:** S3/Swift API calls fail, or Ceph Object Gateway (RGW) returns 5xx errors.

**Step 1 — Check RGW daemon status:**
```bash
ceph orch ps | grep rgw

# Direct service check:
curl -s http://<rgw-host>:7480/
# Should return an XML response; connection refused means daemon is down
```

**Step 2 — If the daemon is down:**
```bash
# Cephadm:
ceph orch daemon restart rgw.<id>

# Systemd:
systemctl restart ceph-radosgw@rgw.<zone>.service
journalctl -u ceph-radosgw@rgw.default --since "30 minutes ago" | tail -50
```

**Step 3 — If the daemon is running but returning errors:**
```bash
# Check RGW log for request errors:
tail -100 /var/log/ceph/ceph-client.rgw.*.log
# or:
cephadm logs --name rgw.<id> | tail -100

# Check the RGW metadata pool:
ceph health detail | grep -i rgw
rados -p .rgw.root ls | head -10
```

**Step 4 — If RGW can't reach OSDs (cluster healthy but RGW gets I/O errors):**
```bash
# Verify the RGW keyring has correct caps:
ceph auth get client.rgw.<id>
# Expected caps: mon 'allow rw', osd 'allow rwx', mgr 'allow rw'

# Test direct RADOS access:
rados -p default.rgw.meta ls --max-objects 5
```

---

## Log Locations

| Component | Default Log Path | Cephadm (containerized) |
|-----------|-----------------|-------------------------|
| OSD | `/var/log/ceph/ceph-osd.<id>.log` | `cephadm logs --name osd.<id>` |
| MON | `/var/log/ceph/ceph-mon.<id>.log` | `cephadm logs --name mon.<id>` |
| MGR | `/var/log/ceph/ceph-mgr.<id>.log` | `cephadm logs --name mgr.<id>` |
| MDS | `/var/log/ceph/ceph-mds.<id>.log` | `cephadm logs --name mds.<id>` |
| RGW | `/var/log/ceph/ceph-client.rgw.*.log` | `cephadm logs --name rgw.<id>` |
| Cluster-wide | `ceph log last 100` | same |

For all file-based logs:
```bash
ls /var/log/ceph/
# Files named: ceph-<daemon-type>.<id>.log
# Rotated files: ceph-osd.3.log.1, ceph-osd.3.log.2.gz, ...
```

---

## Collecting Debug Info

When a problem cannot be diagnosed from normal health output, raise debug levels and collect extended logs.

**Raise debug level on a running daemon (no restart required):**
```bash
# For a specific monitor (with or without quorum):
ceph tell mon.<id> config set debug_mon 10/10
ceph tell mon.<id> config set debug_ms 1/5

# For all monitors at once:
ceph tell mon.* config set debug_mon 10/10

# For a specific OSD:
ceph tell osd.<id> config set debug_osd 10/10
ceph tell osd.<id> config set debug_ms 1/5

# Without quorum, use the admin socket directly:
ceph daemon mon.<id> config set debug_mon 10/10
ceph daemon osd.<id> config set debug_osd 10/10
```

**Collect a diagnostic bundle for a daemon:**
```bash
# Admin socket help lists all available debug commands:
ceph daemon osd.<id> help

# Dump performance counters:
ceph daemon osd.<id> perf dump

# Dump current config:
ceph daemon osd.<id> config show

# Dump PG states for the OSD:
ceph daemon osd.<id> dump_pgs_brief

# For monitors:
ceph daemon mon.<id> quorum_status
ceph daemon mon.<id> mon_status
```

**Cluster-wide diagnostic snapshot:**
```bash
# Full cluster report (useful for bug reports):
ceph report > /tmp/ceph-report-$(date +%Y%m%d-%H%M%S).json

# OSD map:
ceph osd dump > /tmp/osdmap.txt

# PG map summary:
ceph pg stat
ceph pg dump > /tmp/pgmap.txt

# Crush map:
ceph osd getcrushmap -o /tmp/crushmap.bin
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt
```

**Return debug levels to normal when done:**
```bash
ceph tell mon.* config set debug_mon 1/5
ceph tell osd.* config set debug_osd 1/5
```

---

<!-- Source: Official Ceph documentation
     - doc/rados/troubleshooting/troubleshooting-osd.rst
     - doc/rados/troubleshooting/troubleshooting-mon.rst
     - doc/rados/troubleshooting/troubleshooting-pg.rst
     Accessed from https://github.com/ceph/ceph (main branch, March 2026)
-->
