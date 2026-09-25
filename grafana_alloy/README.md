# grafana_alloy

Installs a **Grafana Alloy** agent and points it at Grafana Cloud: node metrics
by `prometheus.remote_write`, the systemd journal and nginx logs by `loki.write`.

## One per host, installed by whoever owns the host

The agent is **host-wide, not application-wide**. It forwards the whole systemd
journal and the node's metrics — every unit on the box, not one service. So it
is installed **once per machine, by the repository that owns that machine**,
which for Maglia means the environment/substrate repository rather than an
application repository.

This matters on a shared box. Two application repositories installing this role
do *not* double the metrics — there is one `alloy` unit and one config file, so
identical runs converge. What they do is **diverge**: any difference in their
variables makes `/etc/alloy/config.alloy` flap, with whichever provision ran
last winning. Config that changes depending on who deployed most recently is
worse than config with a single owner. The same reasoning already governs
`letsencrypt`'s host-global `cli.ini`.

Applications need do nothing to be observed. Their `systemd` units are picked up
by the journal scrape because it takes the journal, not a service list.

## Variables

Required — the role asserts each is present and not `CHANGE_ME`, and fails the
play rather than starting an agent that ships nowhere:

| | |
| --- | --- |
| `grafana_cloud_prometheus_url` | Stack's Prometheus push endpoint. Not secret. |
| `grafana_cloud_loki_url` | Stack's Loki push endpoint. Not secret. |
| `grafana_cloud_prometheus_username` | Stack's numeric instance ID. |
| `grafana_cloud_loki_username` | Stack's numeric instance ID. |
| `grafana_cloud_api_token` | Access policy token, `metrics:write` + `logs:write`. |
| `environment_name` | Becomes the `environment` label on every metric and log. |

The shard numbers in both URLs differ per stack, so copy them from the Grafana
Cloud console rather than from another stack's configuration.

Optional:

| | default | |
| --- | --- | --- |
| `grafana_alloy_enabled` | `false` | Guard, for callers that gate the import. |
| `grafana_alloy_config_path` | `/etc/alloy/config.alloy` | |
| `grafana_alloy_scrape_interval` | `60s` | |

Give each environment its own token. One token pushing two environments means
rotating either rotates both.

## Usage

```yaml
- name: Configure Grafana Alloy
  import_role:
    name: grafana_alloy
  when: grafana_alloy_enabled | default(false)
```

## What it collects

Node metrics (CPU, memory, disk, filesystem, network), Alloy's own health, the
full systemd journal with `unit` / `boot_id` / `transport` / `level` labels,
`/var/log/syslog` and `/var/log/*.log`, and `/var/log/nginx/*.log`.

nginx logs are readable because the role adds `alloy` to the `adm` group — which
it needs for the journal in any case, and which is exactly what Debian's
logrotate grants on those files (`root:adm 0640`). The glob matches only `*.log`,
so rotated `.log.1` and `.gz` files are not re-ingested each rotation.

## What it does not collect

- **Managed database metrics.** A hosted PostgreSQL or MySQL instance reports to
  its provider's console, not to an agent on an application server.
- **Application metrics.** Nothing is scraped from the applications themselves;
  they would each need a Prometheus endpoint and a `prometheus.scrape` block.
- **Blackbox / uptime.** Nothing polls a URL from outside. An agent on the box
  cannot tell you the box is unreachable.
