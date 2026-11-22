---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Server Ops Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```bash\nset -eu\nx=5\n[ \"$x\" -gt 3 ] && echo big || echo small\nname=\"\"\necho \"${name:-anon}\"\nn=0; for i in 1 2 3; do n=$((n + i)); done; echo $n\n```\n",
    "options": [
      "small\nanon\n6",
      "big\nanon\n6",
      "big\n\n6",
      "big\nanon\n3"
    ],
    "correctIndex": 1,
    "explanation": "${var:-default} substitutes when empty; arithmetic expansion sums the loop. Verified by running it in bash."
  },
  {
    "question": "What does this print?\n```bash\nprintf 'a 1\\nb 2\\na 3\\n' | awk '{s[$1]+=$2} END {for (k in s) print k, s[k]}' | sort\n```\n",
    "options": [
      "a 4\nb 4",
      "a 3\nb 2",
      "a 1\nb 2",
      "a 4\nb 2"
    ],
    "correctIndex": 3,
    "explanation": "awk accumulates per key. Verified by running it in bash."
  },
  {
    "question": "What does this print?\n```bash\nset -e\nf() { echo start; false; echo not-reached; }\nf || echo handled\necho after\n```\n",
    "options": [
      "start\nafter",
      "handled\nafter",
      "start\nhandled\nafter",
      "start\nnot-reached\nafter"
    ],
    "correctIndex": 3,
    "explanation": "A well-known gotcha: set -e is ignored inside a function that is called as part of an || list, so false does not stop f, f returns the status of its last command (0) and handled never prints. Verified by running it in bash."
  },
  {
    "question": "How many minutes of downtime does a 99.95% monthly SLA allow in 30 days? (1 decimal)\n```python\nprint(round(30*24*60*(1-0.9995),1))\n```\n",
    "options": [
      "2.2",
      "43.2",
      "4.3",
      "21.6"
    ],
    "correctIndex": 3,
    "explanation": "43,200 minutes x 0.05% = 21.6. Verified by running it."
  },
  {
    "question": "A service needs 450 req/s peak; each instance handles 60 req/s. With N+1 how many instances? (round up for peak, then add one)\n```python\nimport math\nprint(math.ceil(450/60)+1)\n```\n",
    "options": [
      "9",
      "8",
      "7",
      "10"
    ],
    "correctIndex": 0,
    "explanation": "8 for peak (7.5 rounded up) plus one spare. Verified by running it."
  },
  {
    "question": "You disable SSH password login and close your only session; key login then fails. What should you have done?",
    "options": [
      "Nothing, it cannot happen",
      "Restart the server",
      "Test key login in a second session first, validate with sshd -t and keep the first session open",
      "Use telnet"
    ],
    "correctIndex": 2,
    "explanation": "Lock-out is the classic hardening mistake."
  },
  {
    "question": "A deploy script fails halfway and is re-run. What property makes that safe?",
    "options": [
      "Comments",
      "Speed",
      "Idempotency: every step checks state first so re-running reaches the same end state",
      "Using sudo"
    ],
    "correctIndex": 2,
    "explanation": "mkdir -p, ln -sfn, check-then-append."
  },
  {
    "question": "Why is renaming a symlink over 'current' used for deploys?",
    "options": [
      "It compresses files",
      "It encrypts releases",
      "It is faster to type",
      "It switches releases atomically, so requests never see a half-deployed directory"
    ],
    "correctIndex": 3,
    "explanation": "Rollback is the same swap in reverse."
  },
  {
    "question": "Your secrets file was committed to git by mistake. What is the correct response?",
    "options": [
      "Make the repo private",
      "Delete the commit only",
      "Rotate the exposed secrets immediately; removing the commit is not enough",
      "Force-push and carry on"
    ],
    "correctIndex": 2,
    "explanation": "Assume exposure."
  },
  {
    "question": "Why should the same build artifact go to staging and production?",
    "options": [
      "Faster CI",
      "Law",
      "Saves disk",
      "What was tested is exactly what ships; only configuration differs"
    ],
    "correctIndex": 3,
    "explanation": "Build once, promote."
  },
  {
    "question": "What does a blue-green deploy let you do that an in-place deploy does not?",
    "options": [
      "Skip tests",
      "Avoid load balancers",
      "Test the new version fully before real traffic and roll back instantly by switching back",
      "Use less capacity"
    ],
    "correctIndex": 2,
    "explanation": "Cost: duplicate capacity."
  },
  {
    "question": "Which alert is better: 'any 5xx' or 'error rate above 5% for 5 minutes'?",
    "options": [
      "Any 5xx",
      "Both equally",
      "The sustained rate: it indicates ongoing user impact and avoids paging on noise",
      "Neither"
    ],
    "correctIndex": 2,
    "explanation": "Alert fatigue hides real incidents."
  },
  {
    "question": "What is the RPO of a nightly backup at best?",
    "options": [
      "1 hour",
      "1 minute",
      "Up to 24 hours of data loss",
      "Zero"
    ],
    "correctIndex": 2,
    "explanation": "RPO is bounded by backup frequency."
  },
  {
    "question": "Why test restores rather than only checking that backups ran?",
    "options": [
      "Cost",
      "Law",
      "A backup that was never restored is unproven",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "Restore drills find broken backups."
  },
  {
    "question": "Two independent web servers sit behind one load balancer and one database. What is the single point of failure?",
    "options": [
      "The web servers",
      "The load balancer and the database (unless they are redundant too)",
      "Nothing",
      "The network cable only"
    ],
    "correctIndex": 1,
    "explanation": "Redundancy must exist at every layer."
  },
  {
    "question": "What is configuration drift and how do you detect it?",
    "options": [
      "Reality diverging from declared config; detect with terraform plan or Ansible --check --diff on a schedule",
      "Clock skew",
      "A bug in cron",
      "Disk fragmentation"
    ],
    "correctIndex": 0,
    "explanation": "Alert on unexpected diffs."
  },
  {
    "question": "What is the first priority during an incident?",
    "options": [
      "Find who to blame",
      "Write the report",
      "Mitigate user impact, then diagnose the root cause",
      "Restart everything"
    ],
    "correctIndex": 2,
    "explanation": "Stop the bleeding first."
  },
  {
    "question": "Why run containers as non-root and keep state outside them?",
    "options": [
      "Disk space",
      "Limits compromise impact and keeps containers disposable and replaceable",
      "Colour",
      "Speed"
    ],
    "correctIndex": 1,
    "explanation": "Volumes or external services hold state."
  },
  {
    "question": "What belongs in a runbook?",
    "options": [
      "Blame assignments",
      "Org charts",
      "Meeting notes",
      "Symptoms, checks, safe fix steps, escalation contacts and how to verify recovery"
    ],
    "correctIndex": 3,
    "explanation": "Written before the incident."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Atomic releases with health check and rollback

Write a bash release manager: each release is a directory, 'current' is a symlink swapped atomically, a failed health check rolls back automatically, and old releases are pruned. It runs in a temp directory so it needs no root and touches no real server; real deployments also need service restarts and drained connections.

**Starter code**

```bash
#!/usr/bin/env bash
# Atomic release switching with health check and automatic rollback (runs in a temp dir, no root needed)
set -euo pipefail
ROOT=$(mktemp -d)
mkdir -p "$ROOT/releases"

# TODO: switch_to <version>   point "$ROOT/current" at the release by creating a fresh symlink and renaming it over "current"
#       (rename is atomic; BSD mv uses -h, GNU mv uses -T)
# TODO: deploy <version> <healthy|broken>   create the release (write the mode to STATUS), remember the previous release,
#       switch, then "health check" by reading current/STATUS; if it is not "healthy" print "deploy <v>: health check FAILED",
#       switch back to the previous release and print "rolled back to <prev>", and return 1; otherwise print "deploy <v>: ok"
# TODO: prune <n>   keep the newest n releases (sorted by name), printing "pruned <name>" for each one removed (portable: no head -n -N)

deploy v1 healthy
deploy v2 healthy
deploy v3 broken || true
echo "current -> $(basename "$(readlink "$ROOT/current")")"
deploy v4 healthy
prune 2
echo "releases: $(ls -1 "$ROOT/releases" | tr '\n' ' ')"
echo "current -> $(basename "$(readlink "$ROOT/current")")"
```

**Expected output**

```text
deploy v1: ok
deploy v2: ok
deploy v3: health check FAILED
rolled back to v2
current -> v2
deploy v4: ok
pruned v1
pruned v2
releases: v3 v4 
current -> v4
```

<details><summary>Show solution and explanation</summary>

<pre><code>#!/usr/bin/env bash
# Atomic release switching with health check and automatic rollback (runs in a temp dir, no root needed)
set -euo pipefail
ROOT=$(mktemp -d)
mkdir -p &quot;$ROOT/releases&quot;

switch_to() {                    # point &quot;current&quot; at a release by renaming a fresh symlink over it
  ln -sfn &quot;$ROOT/releases/$1&quot; &quot;$ROOT/current.tmp&quot;
  mv -fh &quot;$ROOT/current.tmp&quot; &quot;$ROOT/current&quot; 2&gt;/dev/null || mv -fT &quot;$ROOT/current.tmp&quot; &quot;$ROOT/current&quot;   # BSD -h, GNU -T
}

deploy() {                       # deploy &lt;version&gt; &lt;healthy|broken&gt;
  local ver=$1 mode=$2 prev=&quot;&quot;
  mkdir -p &quot;$ROOT/releases/$ver&quot;
  echo &quot;$mode&quot; &gt; &quot;$ROOT/releases/$ver/STATUS&quot;
  [ -L &quot;$ROOT/current&quot; ] &amp;&amp; prev=$(basename &quot;$(readlink &quot;$ROOT/current&quot;)&quot;)
  switch_to &quot;$ver&quot;
  if [ &quot;$(cat &quot;$ROOT/current/STATUS&quot;)&quot; != healthy ]; then
    echo &quot;deploy $ver: health check FAILED&quot;
    if [ -n &quot;$prev&quot; ]; then switch_to &quot;$prev&quot;; echo &quot;rolled back to $prev&quot;; fi
    return 1
  fi
  echo &quot;deploy $ver: ok&quot;
}

prune() {                        # keep the newest $1 releases (portable: no head -n -N)
  ls -1 &quot;$ROOT/releases&quot; | sort | awk -v keep=&quot;$1&quot; &#x27;{a[NR]=$0} END {for (i = 1; i &lt;= NR - keep; i++) print a[i]}&#x27; |
  while read -r old; do rm -rf &quot;$ROOT/releases/$old&quot;; echo &quot;pruned $old&quot;; done
}

deploy v1 healthy
deploy v2 healthy
deploy v3 broken || true
echo &quot;current -&gt; $(basename &quot;$(readlink &quot;$ROOT/current&quot;)&quot;)&quot;
deploy v4 healthy
prune 2
echo &quot;releases: $(ls -1 &quot;$ROOT/releases&quot; | tr &#x27;\n&#x27; &#x27; &#x27;)&quot;
echo &quot;current -&gt; $(basename &quot;$(readlink &quot;$ROOT/current&quot;)&quot;)&quot;
</code></pre>

<p>Creating the new symlink beside &#x27;current&#x27; and renaming it over the old one is atomic, so readers never see a half-switched state; rollback is the same swap. Note that pruning by name keeps the broken v3 here, which is why real pipelines also delete or mark failed releases. Verified by running the script in bash on macOS.</p>
</details>

### An access-log health report

Parse a small access log with awk: count statuses, compute the server-error rate, find the slowest path by average latency and decide whether to raise an alert. The log is generated inside the script, so no real server logs are used.

**Starter code**

```bash
#!/usr/bin/env bash
# Access-log health report: status counts, error rate, slowest path and an alert decision
set -euo pipefail
LOG=$(mktemp)
cat > "$LOG" <<'LOGS'
10.0.0.1 GET /home 200 35
10.0.0.2 GET /api/items 200 120
10.0.0.3 POST /api/pay 500 910
10.0.0.1 GET /home 200 40
10.0.0.4 GET /api/items 503 30
10.0.0.2 POST /api/pay 500 880
10.0.0.5 GET /missing 404 12
10.0.0.3 GET /api/items 200 150
10.0.0.9 GET /home 200 33
10.0.0.6 POST /api/pay 200 420
LOGS

# TODO: print the status counts (sorted), then the totals line, the slowest path by average latency and the alert decision:
#   status counts:
#   <status> <count> ...
#   requests=<n> server_errors=<5xx count> rate=<percent with 1 decimal>%
#   slowest=<path> avg=<rounded>ms
#   ALERT: error rate above 15%   (or OK when the rate is 15% or less)
# Columns are: client method path status milliseconds
```

**Expected output**

```text
status counts:
200 6
404 1
500 2
503 1
requests=10 server_errors=3 rate=30.0%
slowest=/api/pay avg=737ms
ALERT: error rate above 15%
```

<details><summary>Show solution and explanation</summary>

<pre><code>#!/usr/bin/env bash
# Access-log health report: status counts, error rate, slowest path and an alert decision
set -euo pipefail
LOG=$(mktemp)
cat &gt; &quot;$LOG&quot; &lt;&lt;&#x27;LOGS&#x27;
10.0.0.1 GET /home 200 35
10.0.0.2 GET /api/items 200 120
10.0.0.3 POST /api/pay 500 910
10.0.0.1 GET /home 200 40
10.0.0.4 GET /api/items 503 30
10.0.0.2 POST /api/pay 500 880
10.0.0.5 GET /missing 404 12
10.0.0.3 GET /api/items 200 150
10.0.0.9 GET /home 200 33
10.0.0.6 POST /api/pay 200 420
LOGS

echo &quot;status counts:&quot;
awk &#x27;{c[$4]++} END {for (s in c) print s, c[s]}&#x27; &quot;$LOG&quot; | sort

awk -v limit=15 &#x27;
  {total++; if ($4 &gt;= 500) err++; sum[$3] += $5; n[$3]++}
  END {
    rate = 100 * err / total
    printf &quot;requests=%d server_errors=%d rate=%.1f%%\n&quot;, total, err, rate
    worst = &quot;&quot;; wavg = 0
    for (p in sum) { avg = sum[p] / n[p]; if (avg &gt; wavg) { wavg = avg; worst = p } }
    printf &quot;slowest=%s avg=%.0fms\n&quot;, worst, wavg
    print (rate &gt; limit ? &quot;ALERT: error rate above &quot; limit &quot;%&quot; : &quot;OK&quot;)
  }&#x27; &quot;$LOG&quot;
</code></pre>

<p>One awk pass accumulates totals and per-path sums, then END computes the rate and picks the path with the highest average. Alerting on a rate over a threshold is the same idea as the sustained-error alert in the monitoring lessons. Verified by running the script in bash on macOS.</p>
</details>
