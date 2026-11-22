# 03. Single-Server Setup (Lab / Small Setup)

Goal: run **Elasticsearch + Kibana + Logstash + Filebeat** on **one Ubuntu server** and see logs in Kibana.

> [!WARNING]
> One server = **no high availability**. If it dies, you lose the stack. Fine for learning, a PoC, or a small internal tool. For production use [05](05-multi-server-production-setup.md).

```mermaid
flowchart LR
    F["Filebeat"] --> L["Logstash :5044<br/>(optional)"] --> E[("Elasticsearch :9200<br/>127.0.0.1 only")] --> K["Kibana :5601"]
    F -. "or direct" .-> E
```

---

## ✅ Requirements

| Item | Minimum (lab) | Recommended |
|------|---------------|-------------|
| OS | Ubuntu 22.04 / 24.04 LTS | Same |
| RAM | **4 GB** (Elasticsearch + Kibana only) | **8 GB** (with Logstash too) |
| CPU | 2 vCPU | 4 vCPU |
| Disk | 30 GB SSD | SSD / NVMe, sized by log volume |
| Network | Ports only open as needed | Private network, firewall on |

**Why RAM matters:** Elasticsearch runs on the JVM. Give it a **heap of about 50% of RAM**, never more than ~31 GB. The other 50% is used by the OS file cache, which makes searches fast.

> [!NOTE]
> **No need to install Java.** Elasticsearch and Logstash ship with a bundled JDK. Installing your own is only for special cases.

---

## Step 1: Prepare the OS

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https curl gnupg ca-certificates

# Accurate time is critical for logs
sudo timedatectl set-ntp true
timedatectl status
```

Disable swap (swapping destroys Elasticsearch performance):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Check the memory-map limit (the DEB package normally sets this for you):

```bash
sysctl vm.max_map_count
# should be 262144 or higher; if not:
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl --system
```

---

## Step 2: Add the official Elastic APT repository

> Use the **official Elastic repo**, not a GitHub clone. It gives signed packages and easy upgrades with `apt`.

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-9.x.list

sudo apt update
```

(Use `8.x` instead of `9.x` if you want the 8 series.)

---

## Step 3: Install and configure Elasticsearch

```bash
sudo apt install -y elasticsearch
```

> [!IMPORTANT]
> At the end of the install, the terminal prints the **password for the `elastic` superuser** (shown **once**). Copy it now. If you lose it, reset it later (see Step 8).

The installer automatically: enables security, generates TLS certificates, and configures a **single-node cluster**.

### 3.1 Set the JVM heap

Create a drop-in file (never edit `jvm.options` directly; it is overwritten on upgrades):

```bash
sudo tee /etc/elasticsearch/jvm.options.d/heap.options > /dev/null << 'HEAP'
-Xms2g
-Xmx2g
HEAP
```

Rules: `Xms` = `Xmx` (same value), at most 50% of RAM, at most ~31 GB. (2 GB suits a 4-8 GB lab box.)

### 3.2 Edit `/etc/elasticsearch/elasticsearch.yml`

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
cluster.name: elk-lab
node.name: node-1

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Listen only on localhost: nothing outside this machine can reach it
network.host: 127.0.0.1
http.port: 9200
```

**Why `127.0.0.1`?** `network.host` decides which network interface Elasticsearch listens on.

| Value | Meaning | When |
|-------|---------|------|
| `127.0.0.1` | Only this machine | ✅ Everything on one server (safest) |
| `10.0.0.11` (private IP) | Reachable inside your private network | ✅ Multi-server production |
| `0.0.0.0` | All interfaces, including public | ❌ Almost never; this is how data leaks happen |

> [!WARNING]
> Do **not** add `discovery.type: single-node` if the installer already wrote `cluster.initial_master_nodes`; the two settings conflict and Elasticsearch will refuse to start. The default install already works as a single node.

### 3.3 Start it and enable on boot

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
sudo systemctl status elasticsearch --no-pager
```

(`enable` = start on every reboot; `--now` = also start right now.) Startup can take 30-60 seconds.

### 3.4 Test it

Security is on, so it is **HTTPS + password**:

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic https://localhost:9200
```

Enter the password. Expect JSON with `"tagline" : "You Know, for Search"`.

Check cluster health:

```bash
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic \
  "https://localhost:9200/_cluster/health?pretty"
```

🟡 `yellow` on a single node is normal (replicas have no second node to live on).

---

## Step 4: Install and configure Kibana

```bash
sudo apt install -y kibana
```

### 4.1 Edit `/etc/kibana/kibana.yml`

```yaml
server.port: 5601
server.host: "127.0.0.1"      # keep private; expose via reverse proxy or SSH tunnel
server.name: "kibana-lab"
# server.publicBaseUrl: "https://kibana.example.com"   # set when using a reverse proxy
```

### 4.2 Generate encryption keys (production habit)

These protect saved objects and alerting secrets:

```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
```

Paste the three printed lines (`xpack.encryptedSavedObjects.encryptionKey`, etc.) into `kibana.yml`. **Back them up**; losing them breaks saved alert rules and connectors.

### 4.3 Connect Kibana to Elasticsearch (enrollment token)

```bash
# 1) Create a token (valid ~30 minutes)
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

# 2) Give it to Kibana
sudo /usr/share/kibana/bin/kibana-setup --enrollment-token <PASTE_TOKEN>
```

### 4.4 Start and enable

```bash
sudo systemctl enable --now kibana
sudo systemctl status kibana --no-pager
sudo journalctl -u kibana -f      # watch logs; Ctrl+C to exit
```

### 4.5 Open Kibana safely

Kibana listens on localhost only, so use an **SSH tunnel** from your laptop:

```bash
ssh -L 5601:localhost:5601 youruser@YOUR_SERVER_IP
```

Then open **http://localhost:5601** in your browser.

Login: user **`elastic`**, password from Step 3. Forgot it?

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

> [!TIP]
> **Production access:** put Nginx in front with HTTPS (Let's Encrypt), then set `server.publicBaseUrl`. Minimal Nginx example:
> ```nginx
> server {
>     listen 443 ssl;
>     server_name kibana.example.com;
>     ssl_certificate     /etc/letsencrypt/live/kibana.example.com/fullchain.pem;
>     ssl_certificate_key /etc/letsencrypt/live/kibana.example.com/privkey.pem;
>     location / {
>         proxy_pass http://127.0.0.1:5601;
>         proxy_set_header Host $host;
>         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
>         proxy_set_header X-Forwarded-Proto $scheme;
>     }
> }
> ```
> Restrict by IP / VPN as well.

---

## Step 5: (Optional) Install Logstash

Skip this step if you only need standard logs. Do it when you need custom parsing.

```bash
sudo apt install -y logstash
```

Limit its heap on a small box: edit `/etc/logstash/jvm.options.d/heap.options`:

```text
-Xms1g
-Xmx1g
```

Let Logstash trust Elasticsearch's certificate:

```bash
sudo mkdir -p /etc/logstash/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/logstash/certs/
sudo chown -R logstash:logstash /etc/logstash/certs
```

Create a writer user and role (Kibana → **Dev Tools**):

```json
POST _security/role/logstash_writer
{
  "cluster": ["monitor", "manage_index_templates", "read_ilm"],
  "indices": [
    { "names": ["logs-*", "filebeat-*"],
      "privileges": ["auto_configure", "create_doc", "create_index", "view_index_metadata"] }
  ]
}

POST _security/user/logstash_internal
{
  "password": "CHANGE_ME_TO_A_LONG_RANDOM_PASSWORD",
  "roles": ["logstash_writer"]
}
```

> Never let apps or Logstash use the `elastic` superuser in production: one leaked password = full control. This role is a starting point; tighten it using the docs for your version.

Store the password in the Logstash **keystore** (not in plain text):

```bash
sudo -u logstash /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash create
sudo -u logstash /usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add ES_PWD
```

Create `/etc/logstash/conf.d/beats-pipeline.conf`:

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  # Example: custom app log line
  # 2026-11-22 10:15:32 ERROR payment-service Timeout calling gateway
  # Only parse events that no Filebeat module has already parsed
  if ![event][module] {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:log_level} %{DATA:service_name} %{GREEDYDATA:log_message}" }
      tag_on_failure => ["_grokparsefailure_app"]
    }
    date {
      match  => ["log_time", "yyyy-MM-dd HH:mm:ss"]
      target => "@timestamp"
    }
  }
}

output {
  elasticsearch {
    hosts    => ["https://127.0.0.1:9200"]
    user     => "logstash_internal"
    password => "${ES_PWD}"
    ssl_certificate_authorities => ["/etc/logstash/certs/http_ca.crt"]
    data_stream => true
  }
}
```

> Option names (like `ssl_certificate_authorities`) change between plugin versions. If Logstash complains, check the docs for your installed version.

Test the config, then start:

```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
sudo systemctl enable --now logstash
sudo journalctl -u logstash -f
```

---

## Step 6: Install and configure Filebeat

```bash
sudo apt install -y filebeat
```

### 6.1 Enable ready-made modules (parsers)

```bash
sudo filebeat modules list
sudo filebeat modules enable system nginx     # pick what exists on your server
```

Edit `/etc/filebeat/modules.d/system.yml` and set `enabled: true` for `syslog` and `auth` (set `var.paths` if your distro uses non-default paths).

### 6.2 Edit `/etc/filebeat/filebeat.yml`

```yaml
filebeat.inputs:
  - type: filestream            # current input type (the old "log" input is deprecated)
    id: myapp-logs
    enabled: true
    paths:
      - /var/log/myapp/*.log

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true

name: "lab-server-01"           # appears as host.name
fields:
  environment: lab
fields_under_root: true

# ---- OUTPUT: choose exactly ONE ----

# (A) Direct to Elasticsearch (simplest)
output.elasticsearch:
  hosts: ["https://127.0.0.1:9200"]
  username: "elastic"           # lab only! use an API key in production (see doc 05)
  password: "YOUR_PASSWORD"
  ssl.certificate_authorities: ["/etc/elasticsearch/certs/http_ca.crt"]

# (B) Via Logstash: comment out (A) and use this instead
# output.logstash:
#   hosts: ["127.0.0.1:5044"]
```

### 6.3 Load templates and dashboards, then start

```bash
sudo filebeat test config
sudo filebeat test output
sudo filebeat setup -e           # loads index template + ready-made Kibana dashboards
sudo systemctl enable --now filebeat
sudo systemctl status filebeat --no-pager
```

> If you use output (B) Logstash, run `setup` while pointing at Elasticsearch/Kibana directly:
> `sudo filebeat setup -e -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["https://127.0.0.1:9200"]' -E output.elasticsearch.username=elastic -E output.elasticsearch.password=YOUR_PASSWORD -E output.elasticsearch.ssl.certificate_authorities=/etc/elasticsearch/certs/http_ca.crt`

> [!NOTE]
> Reading `/var/log/*` usually needs root or membership of the `adm` group. The Filebeat package runs as root by default, so this works out of the box.

---

## Step 7: Generate a test log and verify

```bash
logger "ELK test message from lab-server-01"
sleep 5
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic \
  "https://localhost:9200/_cat/indices?v"
```

You should see a `.ds-filebeat-*` or `.ds-logs-*` data stream index with growing `docs.count`. Now go to [04. Kibana hands-on](04-kibana-hands-on.md).

---

## Step 8: Firewall (do not skip)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 443/tcp          # only if you expose Kibana through Nginx
sudo ufw enable
sudo ufw status verbose
```

Do **not** open 9200 or 5601 to the internet.

---

## 📋 Quick reference

| Component | Config file | Service | Port |
|-----------|-------------|---------|------|
| Elasticsearch | `/etc/elasticsearch/elasticsearch.yml` | `elasticsearch` | 9200 (HTTP), 9300 (nodes) |
| Kibana | `/etc/kibana/kibana.yml` | `kibana` | 5601 |
| Logstash | `/etc/logstash/conf.d/*.conf` | `logstash` | 5044 (Beats in) |
| Filebeat | `/etc/filebeat/filebeat.yml` | `filebeat` | none (outbound only) |

➡️ Next: [04. Kibana hands-on](04-kibana-hands-on.md)
