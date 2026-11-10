# PHP-FPM Pool Simulator

An interactive simulator and a short guide to understand how **Apache** works with **PHP-FPM**, and how to tune the PHP-FPM pool.

**Live demo:** open `index.html` in a browser, or visit the GitHub Pages link of this repo.

The simulator is one static HTML file. It has no dependencies, works offline, and supports light and dark mode.

---

## Table of contents

1. [Using the simulator](#using-the-simulator)
2. [How Apache connects to PHP-FPM](#1-how-apache-connects-to-php-fpm)
3. [The FastCGI protocol](#2-the-fastcgi-protocol)
4. [Pool modes (pm)](#3-pool-modes-pm)
5. [Pool settings](#4-pool-settings)
6. [How to size max_children](#5-how-to-size-max_children)
7. [Multiple pools](#6-multiple-pools-one-per-site)
8. [Monitoring with Prometheus and Grafana](#7-monitoring-with-prometheus-and-grafana)
9. [Run it yourself](#run-it-yourself)

---

## Using the simulator

One tick = one second. Each box is a worker:

- **Top number:** how many requests the worker has handled
- **Bottom number:** its RAM

| Color | Meaning |
|---|---|
| Yellow | Busy worker |
| Green | Idle (spare) worker, waiting for a job |
| Purple | Worker that was just replaced because of `max_requests` |
| Dashed | Free slot (not running) |

### Experiments to try

1. **`max_spare_servers`:** set visitors to 15, then drop to 2. PHP-FPM kills one idle worker per second until idle is at or below `max_spare_servers`.
2. **`min_spare_servers`:** set it to 6 and visitors to 0. PHP-FPM keeps 6 workers ready.
3. **Memory leak:** set the leak to 2 MB and `max_requests` to off. RAM keeps growing. Then set `max_requests` to 20: workers restart and RAM goes down.
4. **Traffic spike:** when all workers are busy, the queue grows and the `max_children` warning appears.
5. **Wrong config:** set `min_spare_servers` above `max_spare_servers` to see the error that real PHP-FPM gives.

> **Simplified model.** `max_requests` is small in the simulator so restarts are easy to see (use 500 to 1000 in real life). Real PHP-FPM starts workers in a growing burst (1, 2, 4… up to 32 per second), and real workers handle many requests per second.

---

## 1. How Apache connects to PHP-FPM

Apache does not run PHP by itself. It serves static files (CSS, JS, images) directly and sends `.php` requests to PHP-FPM.

```
Browser ──► Apache ──(FastCGI over socket)──► PHP-FPM master ──► worker ──► database
        ◄──        ◄────────── HTML ─────────                  ◄──
```

1. The browser asks for `index.php`.
2. Apache uses `mod_proxy_fcgi` to forward the request with the FastCGI protocol.
3. The request goes through a **Unix socket** (e.g. `/run/php/php8.3-fpm.sock`) or a **TCP port** (e.g. `127.0.0.1:9000`).
4. The PHP-FPM master process gives the job to a free worker.
5. The worker runs the PHP code (and talks to the database if needed).
6. The worker sends the result back, and Apache sends it to the browser.

### Apache config

```apache
<FilesMatch "\.php$">
    SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
</FilesMatch>
```

With a TCP port:

```apache
<FilesMatch "\.php$">
    SetHandler "proxy:fcgi://127.0.0.1:9000"
</FilesMatch>
```

Enable the modules (Debian/Ubuntu):

```bash
a2enmod proxy proxy_fcgi
a2enconf php8.3-fpm
systemctl restart apache2 php8.3-fpm
```

| Unix socket | TCP port |
|---|---|
| Faster, same machine only | A bit slower, works across servers or containers |

**Why PHP-FPM instead of `mod_php`?** With `mod_php`, PHP lives inside every Apache process, even the ones that only serve images. With PHP-FPM, Apache stays light and PHP has its own pool that you can tune.

---

## 2. The FastCGI protocol

Old **CGI** started a new PHP process for every request, which is slow. **FastCGI** keeps PHP processes alive and sends them jobs.

Messages are binary **records**: an 8-byte header (type, request ID, length) plus the content.

| Record | Direction | Meaning |
|---|---|---|
| `BEGIN_REQUEST` | Apache → FPM | Start a new job |
| `PARAMS` | Apache → FPM | Request details: `REQUEST_METHOD`, `SCRIPT_FILENAME`, headers… |
| `STDIN` | Apache → FPM | Request body (POST data) |
| `STDOUT` | FPM → Apache | The answer (HTTP headers + HTML) |
| `END_REQUEST` | FPM → Apache | Job finished |

Key ideas:

- **Persistent:** the processes stay alive between requests.
- **Binary:** faster to parse than text protocols.
- **Same variables as CGI:** `PARAMS` becomes `$_SERVER` in PHP.

> If `SCRIPT_FILENAME` is wrong, you get `File not found` or `Primary script unknown`. This is the most common problem in this setup.

---

## 3. Pool modes (pm)

Pool file (Debian/Ubuntu): `/etc/php/8.3/fpm/pool.d/www.conf`

| Mode | How it works | Good for |
|---|---|---|
| `static` | Always `max_children` workers | Dedicated server, steady traffic |
| `dynamic` | Workers go up and down between limits | Most sites (default) |
| `ondemand` | Zero workers at start, created per request, killed after `process_idle_timeout` | Small servers, many small sites |

---

## 4. Pool settings

| Setting | Meaning |
|---|---|
| `pm.max_children` | Maximum workers at the same time. If all are busy, new requests wait in a queue. Too high = out of RAM. |
| `pm.start_servers` | Workers started when PHP-FPM starts (`dynamic` only). |
| `pm.min_spare_servers` | Minimum idle workers. If fewer, PHP-FPM creates more (`dynamic` only). |
| `pm.max_spare_servers` | Maximum idle workers. If more, PHP-FPM kills the extra ones, about one per second (`dynamic` only). |
| `pm.process_idle_timeout` | Idle time before a worker is killed (`ondemand` only). |
| `pm.max_requests` | Requests a worker handles before it is replaced by a fresh one. Protects against memory leaks. |
| `request_terminate_timeout` | Kills a script that runs too long. |

> **Rule:** `min_spare_servers` ≤ `start_servers` ≤ `max_spare_servers` ≤ `max_children`. If you break it, PHP-FPM will not start.

### `max_spare_servers` in practice

With `max_children = 20`, `min_spare_servers = 3`, `max_spare_servers = 6`, PHP-FPM keeps idle workers between 3 and 6:

| Situation | Busy | Idle | Action |
|---|---|---|---|
| Traffic goes up | 9 | 2 | Creates workers |
| Normal | 9 | 3–6 | Nothing |
| Traffic goes down | 2 | 10 | Kills 4 workers, one per second |

- **Too low:** "flapping": PHP-FPM kills workers, then must create them again for the next wave of traffic.
- **Too high:** wasted RAM on idle workers.
- **A good start:** about 60% of `max_children`.

### `max_requests` in practice

Some code leaks memory, so a worker grows a little after each request:

```
Request 1    → 40 MB
Request 1000 → 120 MB
Request 5000 → 400 MB   ← server runs out of RAM
```

With `pm.max_requests = 500`, the worker is replaced after 500 requests and its RAM goes back to normal. The replacement happens **after** the current request finishes, so visitors see no error.

| Value | Use |
|---|---|
| `0` | Never restart (default). Only if there are no leaks. |
| `200–500` | Leaky code (for example a CMS with many plugins) |
| `500–1000` | Good default |
| `5000+` | Clean, well-tested applications |

Typical starting values for `dynamic`:

```ini
pm = dynamic
pm.max_children = 50
pm.start_servers = 12        ; ~25% of max_children
pm.min_spare_servers = 10    ; ~20% of max_children
pm.max_spare_servers = 30    ; ~60% of max_children
pm.max_requests = 500
```

---

## 5. How to size `max_children`

```
max_children = RAM for PHP / RAM per worker
```

**Example:** 4096 MB total, keep 1000 MB for the OS and the database, so 3000 MB is for PHP. One worker uses 60 MB: 3000 / 60 = 50. Use about 40 to keep a safety margin.

Measure RAM per worker **while the site is busy**:

```bash
# Average RSS of the workers
ps --no-headers -o rss -C php-fpm8.3 | awk '{s+=$1; n++} END {print n, "workers, avg", s/n/1024, "MB"}'

# Biggest worker
ps --no-headers -o rss -C php-fpm8.3 | sort -n | tail -1 | awk '{print $1/1024 " MB"}'

# More exact: PSS counts shared memory correctly
sudo apt install smem
sudo smem -P php-fpm -k -c "pid pss rss"
```

> RSS counts shared memory once per process, so it can look too big. PSS splits shared memory between processes and is closer to the real cost.

Watch workers live:

```bash
watch -n5 "ps -C php-fpm8.3 -o pid,rss,etime --sort=-rss | head"
```

If RAM goes up as workers get older, you have a leak: lower `pm.max_requests`.

If the pool is too small, the log shows:

```
WARNING: [pool www] server reached pm.max_children setting (20), consider raising it
```

---

## 6. Multiple pools (one per site)

One pool per site gives you:

- **Isolation:** a slow or attacked site only uses its own workers.
- **Security:** each pool runs as its own Linux user.
- **Separate tuning:** a big site gets many workers, a small one gets few.

Example `/etc/php/8.3/fpm/pool.d/site-a.conf`:

```ini
[site-a]
user = site-a
group = site-a

listen = /run/php/php8.3-fpm-site-a.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic
pm.max_children = 50
pm.start_servers = 12
pm.min_spare_servers = 10
pm.max_spare_servers = 30
pm.max_requests = 500

php_admin_value[memory_limit] = 256M
php_admin_value[error_log] = /var/log/php/site-a-error.log
```

Example for a small site with `ondemand`:

```ini
[site-b]
user = site-b
group = site-b
listen = /run/php/php8.3-fpm-site-b.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = ondemand
pm.max_children = 5
pm.process_idle_timeout = 10s
```

Apache virtual host:

```apache
<VirtualHost *:443>
    ServerName site-a.example.com
    DocumentRoot /var/www/site-a
    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.3-fpm-site-a.sock|fcgi://localhost"
    </FilesMatch>
</VirtualHost>
```

Notes:

- Each pool needs **its own socket**.
- Apache (`www-data`) must be able to open the socket, or you get `Permission denied` and a 503 error.
- `php_admin_value` cannot be changed by the site's code. `php_value` can.
- The RAM limit is for **all pools together**.

Check:

```bash
php-fpm8.3 -t                      # test the config
sudo systemctl reload php8.3-fpm
ls -l /run/php/                    # one socket per pool
ps aux | grep "php-fpm: pool"      # pool names
```

---

## 7. Monitoring with Prometheus and Grafana

`node_exporter` alone only shows the RAM of the whole server. For PHP-FPM add:

| Tool | Gives you |
|---|---|
| `node_exporter` | Server RAM, CPU, swap |
| `process-exporter` | RAM of the PHP-FPM workers, including PSS |
| `php-fpm_exporter` | Active and idle workers, queue, `max_children reached` |

Enable the status page in the pool:

```ini
pm.status_path = /fpm-status
```

`process-exporter` config:

```yaml
process_names:
  - name: "php-fpm"
    comm:
      - php-fpm8.3
```

Run it as root (or the PHP user) so it can read `/proc/<pid>/smaps` for PSS.

Example queries:

```promql
# Average RAM per worker (PSS)
namedprocess_namegroup_memory_bytes{groupname="php-fpm", memtype="proportionalResident"}
  / namedprocess_namegroup_num_procs{groupname="php-fpm"}

phpfpm_active_processes
phpfpm_idle_processes
phpfpm_listen_queue
increase(phpfpm_max_children_reached[1h])
```

Example alerts:

```yaml
groups:
- name: php-fpm
  rules:
  - alert: PhpFpmMaxChildrenReached
    expr: increase(phpfpm_max_children_reached[10m]) > 0
    labels: {severity: warning}
  - alert: PhpFpmQueue
    expr: phpfpm_listen_queue > 0
    for: 2m
    labels: {severity: critical}
  - alert: LowMemory
    expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.10
    for: 5m
    labels: {severity: critical}
```

> Metric names can differ between exporter versions. Check `/metrics` on each exporter before building dashboards.

Tuning loop: watch for a week, take the worker RAM at the **peak**, recalculate `max_children` with a ~20% margin, change one setting at a time.

---

## Run it yourself

No build step. Just open the file:

```bash
git clone <this-repo>
open index.html        # macOS
xdg-open index.html    # Linux
```

### Host it with GitHub Pages

1. Go to **Settings → Pages**.
2. Source: **Deploy from a branch**, branch `main`, folder `/ (root)`.
3. After a minute or two the simulator is live at `https://<user>.github.io/<repo>/`.

---

## References

- [PHP manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration)
- [Apache mod_proxy_fcgi](https://httpd.apache.org/docs/2.4/mod/mod_proxy_fcgi.html)
- [Tideways: An Introduction to PHP-FPM Tuning](https://tideways.com/profiler/blog/an-introduction-to-php-fpm-tuning)
