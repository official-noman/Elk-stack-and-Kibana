# 07. Pros, Cons, and Pitfalls

An honest look. Good engineers know when **not** to use a tool.

---

## 👍 Pros

| Pro | Detail |
|-----|--------|
| **Very fast search** | Full-text search over huge data in milliseconds |
| **One place for everything** | Logs, metrics, uptime, security events |
| **Scales horizontally** | Add nodes; shards spread automatically |
| **Rich ecosystem** | Hundreds of ready-made modules and integrations (Nginx, MySQL, AWS, Docker, Kubernetes...) |
| **Flexible** | Structured and unstructured data; powerful aggregations for analytics |
| **Great UI** | Kibana dashboards, alerting, maps, machine-learning features (some paid) |
| **Huge community** | Documentation, examples, hiring pool |
| **Open APIs** | Everything is a REST call, easy to automate |

## 👎 Cons

| Con | Detail |
|-----|--------|
| **Resource hungry** | JVM, lots of RAM and fast disk; costs grow with log volume |
| **Operational effort** | Sizing, shard management, upgrades, backups, monitoring |
| **Learning curve** | Mappings, shards, ILM, KQL/Query DSL, Logstash grok |
| **Not a transactional database** | No ACID transactions; do not use as your system of record |
| **Storage cost** | Full indexing makes data bigger than raw logs |
| **Licensing complexity** | Core Elasticsearch/Kibana are source-available with an open-source (AGPL) option; some features need paid tiers; the AWS-led fork **OpenSearch** is Apache 2.0. Check what your use case needs |
| **Cluster failure modes** | Split-brain, red status, unassigned shards, heap pressure |
| **Not instant** | Near-real-time (~1s) |

---

## 🧨 Where to be careful (common production disasters)

| # | Pitfall | What happens | Prevention |
|---|---------|--------------|------------|
| 1 | **Open port 9200** | Anyone reads/deletes your data | Private network, security on, firewall |
| 2 | **No retention policy** | Disk fills → indices go read-only → ingestion stops | ILM from day one, disk alerts at 75% |
| 3 | **Too many small shards** | Slow cluster, heap exhaustion | Rollover by size, template with few shards |
| 4 | **Heap set wrong** | OOM crashes or slow GC | 50% of RAM, max ~31 GB, `Xms = Xmx` |
| 5 | **Swap enabled** | Random huge latency | Disable swap |
| 6 | **2-node "cluster"** | No quorum → split-brain risk | 3 master-eligible nodes |
| 7 | **No backups / untested restore** | Permanent data loss | SLM + restore drills |
| 8 | **Mapping explosion** | Thousands of fields, slow and unstable | Fixed schemas, `flattened`, field limits |
| 9 | **Logging everything at DEBUG** | Surprise 10× cost | Control log levels at the source |
| 10 | **Wrong time / timezone** | Events appear in wrong order | NTP, UTC, `date` filter for custom logs |
| 11 | **Sensitive data in logs** | Privacy/compliance breach | Mask at source or in pipeline |
| 12 | **Version mismatch** | Agents newer than ES won't connect | Same major version, upgrade ES first |
| 13 | **Single Logstash** | Pipeline outage = data delay/loss | 2+ nodes, persistent queues, or Kafka |
| 14 | **Central server on a busy app host** | App and logs die together | Dedicated hosts |
| 15 | **Using `elastic` superuser everywhere** | One leak = total compromise | Roles + API keys |
| 16 | **Never testing failure** | HA that doesn't actually work | Kill a node on purpose in staging |
| 17 | **Lost encryption keys / certs** | Alerts and connectors break | Back up secrets securely |
| 18 | **No ingestion-gap alert** | A broken agent goes unnoticed for weeks | Alert on "no data from host X" |

---

## 🤔 When ELK is a good choice

- You have **many servers/services** and need central search and dashboards
- You need **powerful full-text search** and analytics across logs
- You need **security/SIEM-style** investigation
- You have (or will invest in) the operations skills, or you use a managed service

## 🙅 When to think twice

- **1-3 small servers:** `journalctl`, `grep`, or a simple tool may be enough
- **Tiny budget, huge log volume, mostly "grep by label":** consider Loki (cheaper storage model)
- **No one to operate it:** use a managed service instead of self-hosting
- **You want a transactional store:** use a real database

---

## 🔀 Alternatives at a glance

| Tool | Idea | Notes |
|------|------|-------|
| **Grafana Loki + Grafana** | Index only labels, store compressed logs cheaply | Lower cost/resource; weaker full-text analytics |
| **OpenSearch + Dashboards** | Apache-2.0 fork of Elasticsearch/Kibana | Very similar concepts; AWS-backed |
| **Graylog** | Log management platform (uses OpenSearch/Elasticsearch underneath) | Simpler log-focused UX |
| **Splunk** | Commercial leader | Powerful and expensive |
| **Datadog / New Relic / managed SaaS** | Fully hosted observability | Easy, but per-GB cost adds up |
| **Cloud-native** (CloudWatch, Azure Monitor, GCP Logging) | Built into the cloud | Convenient if you live in one cloud |

> Learning ELK concepts (indexing, shards, pipelines, retention) transfers well to OpenSearch and other tools.

➡️ Next: [08. Troubleshooting cheat sheet](08-troubleshooting-cheatsheet.md)
