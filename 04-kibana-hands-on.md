# 04. Kibana Hands-On

You finished [03](03-single-server-setup.md) and logs are flowing. Now let's use them.

---

## 1. Open Kibana and log in

| Item | Value |
|------|-------|
| URL (SSH tunnel) | `http://localhost:5601` after `ssh -L 5601:localhost:5601 user@SERVER_IP` |
| URL (production) | `https://kibana.example.com` (Nginx + TLS) |
| Username | `elastic` (admin) |
| Password | Printed during Elasticsearch install, or reset it: |

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

First visit may ask for an **enrollment token** and a **6-digit verification code**:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
sudo /usr/share/kibana/bin/kibana-verification-code
```

> [!TIP]
> Do your daily work with a **normal user**, not `elastic`. Create one under **Stack Management → Security → Users** (see Step 8).

---

## 2. Check that data exists

Open the menu (☰) → **Management → Dev Tools** and run:

```text
GET _cat/indices?v
GET _cat/health?v
GET filebeat-*/_count
```

(If you use data streams the names look like `.ds-logs-*` or `.ds-filebeat-*`.) A growing `docs.count` means ingestion works.

---

## 3. Create a Data View

A **Data View** tells Kibana *which indices to read* (like choosing which database table to query).

1. ☰ → **Management → Stack Management → Kibana → Data Views**
2. Click **Create data view**
3. **Name:** `Filebeat logs`
4. **Index pattern:** `filebeat-*` (or `logs-*`). The right side should show matching sources.
5. **Timestamp field:** `@timestamp`
6. Click **Save data view to Kibana**

> [!NOTE]
> Older docs call this an "Index Pattern". Same thing, new name.

---

## 4. Explore in Discover

☰ → **Analytics → Discover**, then select your data view.

| Action | How |
|--------|-----|
| Change time range | Top right picker: "Last 15 minutes", "Last 24 hours", custom |
| Add columns | Hover a field in the left list → ➕ |
| Filter by value | Click a field → ➕ / ➖ magnifier |
| Search | KQL bar (examples below) |

### KQL (Kibana Query Language) cheat sheet

| Goal | Query |
|------|-------|
| Free text | `timeout` |
| Exact field | `host.name : "lab-server-01"` |
| Errors only | `log.level : "error"` |
| AND / OR / NOT | `log.level : "error" and not host.name : "db-01"` |
| Wildcard | `message : *refused*` |
| Numeric range | `http.response.status_code >= 500` |
| Field exists | `user.name : *` |

Try it: run `logger "ELK test message"` on the server, wait 5 seconds, search `message : "ELK test message"`.

---

## 5. Build visualizations (Lens)

☰ → **Analytics → Visualize Library → Create visualization → Lens**

### 5.1 Logs over time (bar chart)

1. Visualization type: **Bar vertical stacked**
2. Drag `@timestamp` to the horizontal axis
3. Vertical axis: **Count of records**
4. Break down by `log.level` or `host.name`
5. **Save** as `Log volume by host`

### 5.2 Error count (big number)

1. Type: **Metric**
2. Primary metric: **Count of records**
3. Query bar: `log.level : "error"`
4. **Save** as `Errors (total)`

### 5.3 Top servers (pie / donut)

1. Type: **Donut**
2. Slice by: **Top values of `host.name`**
3. Size: **Count**
4. **Save** as `Logs share per server`

### 5.4 Top error messages (table)

1. Type: **Table**
2. Rows: **Top values of `message`** (or `error.message`)
3. Metric: **Count**, filter `log.level : "error"`
4. **Save**

---

## 6. Build a Dashboard

1. ☰ → **Analytics → Dashboards → Create dashboard**
2. Click **Add from library**, add your four visualizations
3. Drag and resize panels
4. Top-right: set time range (e.g., **Last 1 hour**) and **auto-refresh** (e.g., every **30 seconds**)
5. **Save** → title `Ops Overview`, enable **Store time with dashboard** if it must always open on the same range

**Instant dashboards:** because you ran `filebeat setup`, Kibana already has ready-made dashboards. Search "dashboards" for `[Filebeat System] ...` or `[Filebeat Nginx] ...`. Use them as a starting point.

---

## 7. Show it on a TV / NOC wall

1. Open the dashboard → **Full screen** button (top bar)
2. Set auto-refresh to 30s or 1m
3. Create a **read-only viewer user** for the screen (Step 8) so nobody can change things
4. Put the dashboard in its own **Space** (e.g., `NOC`)

> [!TIP]
> For a browser kiosk on a TV, log in once with the viewer account. Prefer a dedicated low-privilege user over sharing the admin login.

---

## 8. Create proper users and roles (RBAC)

Principle: **least privilege.**

| Persona | Role | Can do |
|---------|------|--------|
| Admin (you) | `superuser` (few people only) | Everything |
| Developer | Custom: read `logs-*`, use Discover/Dashboards | Search logs, no settings |
| NOC TV | Custom: read-only on `logs-*` + `viewer` Kibana privileges | View dashboards only |
| Shipping agent | API key (not a user) | Write logs only |

Steps:

1. **Stack Management → Security → Roles → Create role**
2. **Indices:** names `logs-*`, `filebeat-*` → privileges `read`, `view_index_metadata`
3. **Kibana → Assign to space:** privilege `Read` (or per feature)
4. **Stack Management → Security → Users → Create user** and assign the role

---

## 9. Create your first alert

1. ☰ → **Stack Management → Rules → Create rule**
2. Choose **Log threshold** (or **Elasticsearch query**)
3. Condition example: **count of `log.level : "error"` is above 50 in the last 5 minutes**, grouped by `host.name`
4. **Action:** pick a connector (Email, Slack, Webhook, Teams...). Create the connector under **Stack Management → Connectors**
5. Message example: `{{rule.name}}: {{context.group}} had {{context.matchingDocuments}} errors`
6. **Save**

> [!NOTE]
> Rules need the Kibana **encryption keys** from doc 03 (Step 4.2). Some connectors and advanced alerting features are **paid-tier**. Check the [Elastic subscriptions page](https://www.elastic.co/subscriptions) before you rely on one.

---

## 10. Hands-on checklist ✅

- [ ] Logged in to Kibana
- [ ] `GET _cat/indices?v` shows growing doc count
- [ ] Created Data View with `@timestamp`
- [ ] Found my test message in Discover using KQL
- [ ] Built at least 3 visualizations
- [ ] Saved a dashboard with auto-refresh
- [ ] Created a non-admin viewer user
- [ ] Created one alert rule and received a notification

➡️ Next: [05. Multi-server production setup](05-multi-server-production-setup.md)
