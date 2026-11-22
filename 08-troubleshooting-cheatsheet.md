# 08. Troubleshooting Cheat Sheet

Rule of thumb: **follow the data path** and find where it stops.

```mermaid
flowchart LR
    A["1. Log file<br/>has new lines?"] --> B["2. Filebeat<br/>reading + sending?"] --> C["3. Network<br/>port reachable?"] --> D["4. Logstash / ES<br/>accepting?"] --> E["5. Index<br/>doc count growing?"] --> F["6. Kibana<br/>data view + time range?"]
```

---

## 🔎 Handy commands

Shorthand: `ES=https://localhost:9200`, `CA=/etc/elasticsearch/certs/http_ca.crt`

```bash
# Service state and logs
sudo systemctl status elasticsearch kibana logstash filebeat --no-pager
sudo journalctl -u elasticsearch -n 100 --no-pager
sudo tail -f /var/log/elasticsearch/<cluster.name>.log

# Cluster
curl --cacert $CA -u elastic "$ES/_cluster/health?pretty"
curl --cacert $CA -u elastic "$ES/_cat/nodes?v"
curl --cacert $CA -u elastic "$ES/_cat/indices?v&s=store.size:desc"
curl --cacert $CA -u elastic "$ES/_cat/shards?v&s=state"
curl --cacert $CA -u elastic "$ES/_cat/allocation?v"
curl --cacert $CA -u elastic "$ES/_cluster/allocation/explain?pretty"

# Filebeat
sudo filebeat test config
sudo filebeat test output
sudo journalctl -u filebeat -f

# Logstash config check
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

# Network from a source server to the central server
nc -vz 10.0.0.10 9200
nc -vz 10.0.0.10 5044
```

---

## 🧯 Symptom → cause → fix

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Elasticsearch won't start; "max virtual memory areas too low" | `vm.max_map_count` too small | `sudo sysctl -w vm.max_map_count=262144` and persist in `/etc/sysctl.d/` |
| Won't start; OOM / heap errors | Heap too big for RAM or too small for load | Set `Xms=Xmx` ≈ 50% RAM in `jvm.options.d/heap.options` |
| Won't start after adding `discovery.type: single-node` | Conflicts with `cluster.initial_master_nodes` | Remove one of them |
| `curl` to 9200 fails / "SSL certificate problem" | Security is on (HTTPS) | Use `https://`, `--cacert http_ca.crt`, and `-u elastic` |
| Cluster **yellow** on one node | Replicas can't be placed on a single node | Normal for lab; set replicas to 0 on dev indices |
| Cluster **yellow/red** on many nodes | Unassigned shards | Run `_cluster/allocation/explain`; check disk, node count, settings |
| Indexing fails: "read-only / disk watermark" | Disk > 95% (flood stage) | Free space or add disk, delete old indices, fix ILM; Elasticsearch releases the block automatically once space is available |
| Kibana "server is not ready yet" | Can't reach Elasticsearch, or still starting | `journalctl -u kibana -f`; check `elasticsearch.hosts`, credentials, CA |
| Kibana login loop / 401 | Wrong password or token expired | Reset password; re-enroll Kibana |
| Alerts/connectors say "encryption key missing" | `xpack.encryptedSavedObjects.encryptionKey` not set | Generate keys, set in `kibana.yml`, restart (back them up!) |
| Filebeat: "connection refused" | Wrong host/port, firewall, service down | `nc -vz host port`, check `ufw`/security group |
| Filebeat: "x509: certificate signed by unknown authority" | CA not trusted | Copy `http_ca.crt`, set `ssl.certificate_authorities` |
| Filebeat: x509 "cannot validate certificate for IP/host" | Cert doesn't include the name you use | Use the name in the cert or reissue a cert with correct SANs |
| Filebeat: 401/403 | Bad API key or missing privileges | Recreate key with required privileges (`create_doc`, `auto_configure`, ...) |
| Filebeat running, no data | Wrong `paths`, permissions, or file is old | `filebeat test config`; check file readable; look for `harvester` messages in logs |
| Old log lines were not re-sent | Filebeat's registry remembers how far it read each file, so it never re-reads old lines | For re-ingestion tests only: stop Filebeat, delete the registry (`/var/lib/filebeat/registry`), start again. Never in production |
| Data in ES but not in Kibana Discover | Wrong **time range** or data view pattern | Widen time range (Last 24h), check `@timestamp`, check data view pattern |
| Timestamps wrong by hours | Timezone / parsing issue | Use UTC, `date` filter in Logstash, check server NTP |
| Fields are all in `message` | No parsing configured | Enable Filebeat module / ingest pipeline / grok |
| `_grokparsefailure` tag | Grok pattern doesn't match | Test patterns in Kibana **Grok Debugger** (Dev Tools) |
| Logstash won't start | Config syntax error | `logstash -t`; read `/var/log/logstash/logstash-plain.log` |
| Logstash: "Pipeline blocked" / slow | Elasticsearch slow or filters expensive | Check ES health, add pipeline workers, simplify grok, use persistent queue |
| Mapping conflict (e.g., field is `long` vs `text`) | Same field name with different types | Fix at source / template; reindex if needed |
| Searches slow | Too many shards, heap pressure, huge time range | Reduce shards, add nodes/RAM, narrow time range, check `_nodes/stats` |

---

## 🧪 Quick test: is the whole pipeline alive?

```bash
# 1) generate a unique line on a source server
logger "PIPELINE_TEST_$(date +%s)"

# 2) wait ~5-10 seconds, then in Kibana Discover search:
#    message : "PIPELINE_TEST_*"
```

If you see it → everything works. If not, walk the path in the diagram at the top of this page.

---

## 🛠️ Reset helpers

```bash
# Reset elastic user password
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic

# New Kibana enrollment token
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

# Verification code shown by Kibana
sudo /usr/share/kibana/bin/kibana-verification-code
```

---

## 📚 Official references

- Elastic documentation: https://www.elastic.co/docs
- Elasticsearch: install, security, ILM, snapshots
- Filebeat reference and module list
- Logstash reference (pipelines, grok patterns)
- Kibana guide (Discover, Lens, Dashboards, Rules)
- Elastic Common Schema (ECS) reference

> When this repo and the official docs disagree, **trust the official docs for your version.**
