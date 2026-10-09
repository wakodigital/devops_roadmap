# Lab 17 — Capstone: a health monitor for the tower

**Uses:** every section of the roadmap
**Time:** 6 to 10 hours, over several sittings
**Touches system state:** yes, and unlike the other labs this one is meant to stay. Names use the prefix `tower-health`. A removal section is at the end.

There are no step-by-step instructions here. You get a specification, acceptance checks you can run, and hints that point back to the lab where each technique was taught. Build it in the order given; each stage works by itself.

When it is finished you will have a small, real piece of operations tooling that you wrote, deployed, scheduled, secured, logged, containerised and can explain line by line.

## What you are building

```
  systemd timer (every 5 min)
        │
        ▼
  tower-health  (bash, runs as an unprivileged user, sandboxed)
        │  collects: load, memory, disk, inodes, failed units, containers,
        │            pending security updates, auth failures, reachability
        ▼
  /var/log/tower-health/health.jsonl   ── logrotate ──▶ compressed history
        │                                                    
        ├──▶ tower-health-report  (awk / jq: trends, worst hour, uptime of checks)
        │
        └──▶ status page container (read-only bind mount, 127.0.0.1 only, limited, hardened)
```

---

## Stage 1 — the collector script

*Labs 05, 06, 14*

Write `tower-health`. Each run appends **one line of JSON** to a log file and prints a short human-readable summary.

Requirements:

- Strict mode, functions, `usage`, `getopts`: `-o FILE` (output file, default `/var/log/tower-health/health.jsonl`), `-q` (no summary), `-n` (dry run: print the JSON, write nothing), `-h`.
- Fields: `ts` (ISO 8601), `host`, `load1`, `ncpu`, `mem_avail_pct`, `swap_used_mb`, `disk_max_pct` and `disk_max_mount` (fullest real filesystem), `inode_max_pct`, `failed_units` (count), `failed_unit_names` (array), `containers_running`, `containers_unhealthy`, `sec_updates`, `auth_failures_1h`.
- Thresholds as variables at the top: load above `ncpu`, memory available below 10%, disk above 85%, any failed unit, any unhealthy container.
- Exit status `0` when everything is within thresholds, `1` when any threshold is breached, `2` for usage errors. The JSON line gains a `status` field (`ok` or `warn`) and a `reasons` array.
- A check that cannot run (for example, no permission to ask Docker) records `null` for its fields and does not abort the script.
- Passes ShellCheck with no warnings.

Hints:

- Build JSON with `jq -n --arg … --argjson …`. Assembling JSON by string concatenation breaks on the first unit name containing a quote.
- `df --output=pcent,target -x tmpfs -x devtmpfs -x overlay` is easier to parse than plain `df`.
- `systemctl --failed --no-legend --plain`, `docker ps --filter health=unhealthy -q`, `dnf updateinfo list --security -q`.
- `dnf` is slow. Cache its answer in a file and refresh it only when the file is more than six hours old (`find FILE -mmin +360`).

Acceptance:

```bash
./tower-health -n | jq .                                   # valid JSON, all fields present
./tower-health -n | jq -e '.status == "ok" or .status == "warn"'
./tower-health -x; echo $?                                 # 2
./tower-health -o /tmp/th.jsonl -q; ./tower-health -o /tmp/th.jsonl -q; wc -l /tmp/th.jsonl      # 2
jq -s 'map(.ts) | length' /tmp/th.jsonl                    # 2
sc tower-health                                            # no output (the ShellCheck function from Lab 14)
```

Then force each threshold in turn (set it absurdly low) and confirm the exit status, `status` and `reasons` change.

---

## Stage 2 — reachability checks

*Labs 13, 15*

Add network fields: `gw_rtt_ms` and `gw_loss_pct` (3 pings to the default gateway), `dns_ms` (time to resolve a name), `http_code` and `http_ms` for a URL given with `-u URL` (a site you run), `tls_days_left` for that URL's certificate.

- Every network check has a timeout. A dead network must make the script slow by seconds, not hang it.
- Breaches: gateway loss above 0%, DNS failure, HTTP code not 2xx or 3xx, certificate expiring within 14 days.

Hints: `ping -c 3 -q -W 1`, `curl -s -o /dev/null -m 10 -w '%{http_code} %{time_total}'`, `dig +tries=1 +time=2`, and for the certificate `curl -sv` does not give a clean date; `openssl s_client -connect HOST:443 -servername HOST </dev/null 2>/dev/null | openssl x509 -noout -enddate` does.

Acceptance: run it with a URL that works, one that returns 404, and a host name that does not resolve. Each must finish within 15 seconds with sensible fields and reasons.

---

## Stage 3 — a dedicated account and correct permissions

*Labs 03, 04, 08*

- Install the script as `/usr/local/bin/tower-health`, root-owned, mode 755.
- Create a system account `towerhealth`: no login shell, no home directory.
- Create `/var/log/tower-health`, owned by `towerhealth`, mode 750, group `adm` so that admins can read it.
- The script must run correctly as `towerhealth`.

The design decision: as an unprivileged user the script cannot read `/var/log/secure`, the full journal, or talk to Docker. Decide how to grant each, and write down why.

| Need | Options | Consider |
|---|---|---|
| read the journal | add the account to `systemd-journal` | read-only, narrowly scoped: reasonable |
| count containers | add to the `docker` group | **root-equivalent** (Lab 08). Too much for a monitor |
| | a sudoers rule for one exact, read-only `docker ps …` command | narrow; mind the argument-matching rules |
| | a second root-run oneshot unit that writes container counts to a file the monitor reads | separates privilege cleanly |
| read `/var/log/secure` | an ACL granting read to the account | survives until logrotate recreates the file: check the `create` line |
| | use the journal instead (`journalctl -u sshd`) | no file permission needed |

Acceptance:

```bash
sudo -u towerhealth /usr/local/bin/tower-health -n | jq '.status, .containers_running, .auth_failures_1h'    # no nulls
sudo -u towerhealth test -w /etc/passwd; echo $?           # 1
id towerhealth                                             # not in docker, not in wheel
sudo -l -U towerhealth                                     # only what you deliberately granted
```

---

## Stage 4 — service, timer, alerting

*Lab 09*

- `tower-health.service`: `Type=oneshot`, runs as `towerhealth`, uses `LogsDirectory=tower-health` (which also creates the directory, so check it agrees with stage 3).
- Exit status 1 means "warning", not "the unit failed". Tell systemd so with `SuccessExitStatus=1`, and decide how a warning should be surfaced instead.
- `tower-health.timer`: every five minutes, `Persistent=true`, a small `RandomizedDelaySec`.
- `tower-health-alert@.service`, triggered through `OnFailure=`, writes a priority `err` line to the journal naming the failed unit. (Later this is where a mail, webhook or push notification would go.)
- Harden the service until `systemd-analyze security tower-health.service` reports a score under 3.0 while it still works. `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `NoNewPrivileges=yes` (which rules out sudo: does that change your stage 3 decision?), an empty `CapabilityBoundingSet=`, `MemoryMax=`, `CPUQuota=`, `Nice=`.

Acceptance:

```bash
systemctl list-timers tower-health.timer                   # scheduled
sudo systemctl start tower-health.service; systemctl status tower-health.service | head -6
sudo journalctl -u tower-health.service -n 5 --no-pager
systemd-analyze security tower-health.service | tail -1    # under 3.0
tail -1 /var/log/tower-health/health.jsonl | jq .ts        # a timestamp from the last five minutes
# break it on purpose (chmod 000 the script), start it, and find the alert:
sudo journalctl -t tower-health-alert -n 1 --no-pager
```

Leave it running for a day before stage 6, so that there is data.

---

## Stage 5 — log rotation and retention

*Lab 12*

- A logrotate stanza: daily, keep 30, compress, `delaycompress`, `missingok`, `notifempty`, `create` with the right owner and mode.
- No `postrotate` is needed. Explain why, in terms of how the script opens the file.
- Work out the disk cost: bytes per line × 288 lines a day × 30 days, uncompressed and compressed.

Acceptance:

```bash
sudo logrotate -d /etc/logrotate.d/tower-health            # parses, and describes what it would do
sudo logrotate -f /etc/logrotate.d/tower-health; ls -l /var/log/tower-health/
sudo systemctl start tower-health.service; ls -l /var/log/tower-health/      # a fresh file, correct owner and mode
```

---

## Stage 6 — reporting

*Lab 05*

Write `tower-health-report`, which reads the current and rotated logs (compressed ones included) and prints:

- the period covered, the number of samples, and the number and percentage with `status == "warn"`
- minimum, average and maximum of `load1`, `mem_avail_pct` and `http_ms`
- the hour of the day with the highest average load
- each distinct warning reason with its count, most frequent first
- gaps: any interval between consecutive samples longer than 11 minutes (the timer missed runs, or the machine was down)

Do it once with `jq` and once with `awk` over `jq -r '[.ts, .load1, …] | @tsv'`. Compare the two for readability.

Acceptance: the report runs in under two seconds on 30 days of data (8,640 lines), and its warning count matches `zcat -f /var/log/tower-health/health.jsonl* | grep -c '"warn"'`.

---

## Stage 7 — a status page in a container

*Labs 13, 16*

Serve the latest sample and the last 24 hours of history as a small web page.

- A tiny application (Python standard library is enough) that reads `health.jsonl` and renders HTML, with `/health` returning 200.
- A Dockerfile: small pinned base, non-root `USER`, exec-form `CMD`, a `HEALTHCHECK`, a SIGTERM handler.
- A `compose.yaml`: the log directory bind-mounted **read-only**; published on `127.0.0.1` only; `read_only: true`; `cap_drop: [ALL]`; `no-new-privileges`; memory, CPU and pids limits; `restart: unless-stopped`; log size capped.
- Reach it from your laptop through an SSH tunnel.

The awkward part is the bind mount. The container user must be able to read the files (mode 640, owner `towerhealth`, group `adm`), and SELinux must allow it, **without** relabelling `/var/log` with `:Z`. Options: run the container with a GID that matches `adm` (`user: "10001:GID"`); have the collector also write a copy to a dedicated directory such as `/srv/tower-health/` that is yours to label; or export with a small root-run unit. Choose one and justify it.

Acceptance:

```bash
docker compose ps                                          # healthy
curl -s http://127.0.0.1:PORT/health                       # ok
docker compose exec status id                              # not uid 0
docker compose exec status sh -c 'echo x >> /data/health.jsonl' ; echo $?      # non-zero: read-only
sudo ss -tlnp | grep PORT                                  # bound to 127.0.0.1
docker stats --no-stream                                   # within its limits
time docker compose stop status                            # well under 10 seconds
ls -Zd /var/log                                            # still var_log_t: you did not relabel it
sudo ausearch -m AVC -ts recent | grep -c tower-health     # 0 denials in normal operation
```

---

## Stage 8 — prove you can rebuild it

*Labs 04, 10, and your Ansible work*

Something that exists only as hand-typed commands on one machine is not finished.

- List every external command the scripts call and the package that provides each (`rpm -qf "$(command -v X)"`). That is the dependency list.
- Make a release tarball containing the scripts, units, logrotate stanza, compose file and a README, with an install script that is safe to run twice.
- Better: write it as an Ansible role (`user`, `file`, `copy`, `template`, `systemd`, `community.docker.docker_compose_v2`), with thresholds and the URL as variables, and handlers for `daemon-reload`. Run it twice and confirm the second run reports no changes.
- Test the role or the install script on a clean Rocky 9 VM or container, not on the tower.

Acceptance: on a fresh machine, one command produces a working timer and the first JSON line within six minutes.

---

## Explain it

When it works, answer these without notes. They are the questions an interviewer asks about a project like this one.

1. Why does the collector run as its own user? What could it do if it ran as root and had a bug in a path variable?
2. What happens, step by step, between the timer firing and a line appearing in the log?
3. Why JSON lines and not a pretty-printed JSON file?
4. Your collector reports `mem_avail_pct`, not free memory. Why?
5. How does the status container read the log without being able to modify it, and which three mechanisms enforce that?
6. The machine was powered off for two hours. What does the timer do when it comes back, and how does the report show the gap?
7. `tower-health.service` shows `status=1/FAILURE`. Is that a bug, or a warning? How did you make systemd tell them apart?
8. The disk filled up anyway. Which of your own components could have caused it, and what limits each?
9. How would you deploy this to five more machines? What changes if they run Ubuntu?
10. Where would this go in Kubernetes terms: which part becomes a CronJob, a ConfigMap, a volume, a Deployment, a probe?

## Removal

```bash
sudo systemctl disable --now tower-health.timer
sudo rm -f /etc/systemd/system/tower-health*.service /etc/systemd/system/tower-health.timer
sudo rm -rf /etc/systemd/system/tower-health*.d
sudo systemctl daemon-reload; sudo systemctl reset-failed
# in the status page's directory:  docker compose down
sudo rm -f /usr/local/bin/tower-health /usr/local/bin/tower-health-report /etc/logrotate.d/tower-health /etc/sudoers.d/*tower-health*
sudo rm -rf /var/log/tower-health /srv/tower-health
sudo userdel towerhealth
```
