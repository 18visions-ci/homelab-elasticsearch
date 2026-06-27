# Enable `xpack.apm_data.enabled` (APM index templates)

**Goal:** make Elasticsearch install the APM index templates (`traces-apm*`,
`metrics-apm*`, `logs-apm*`) so the standalone **homelab-apm-server** can ingest
without Fleet. This is the no-Fleet path required because the cluster runs with
security **disabled** (Fleet needs security enabled).

Pairs with the apm-server side (done separately): apm-server 8.18.1 with
`apm-server.data_streams.wait_for_integration: false`, which skips apm-server's
integration-presence check (that check is broken against apm_data's `@template`
naming — elastic/apm-server#13064).

> Requires an **Elasticsearch restart** — `xpack.apm_data.enabled` is a node
> setting in `elasticsearch.yml`, not a dynamic cluster setting.

---

## The change — already implemented in the role

The setting is now in `roles/elasticsearch_configure/tasks/main.yaml` (a
`lineinfile` task that ensures `xpack.apm_data.enabled: true` in
`/etc/elasticsearch/elasticsearch.yml` and `notify`s the **Restart
Elasticsearch** handler). So you just **re-run the playbook** — no manual edit.

```yaml
xpack.apm_data.enabled: true   # <- what the task writes into elasticsearch.yml
```

---

## Apply (dev first, then prod)

Do **dev (10.7.40.191)** first; verify; then **prod (10.7.30.238)**. The task
changes `elasticsearch.yml` and the handler restarts ES automatically at the end
of the run.

```bash
# Dev
ansible-playbook main.yaml --limit elasticsearch-dev      # adjust to your inventory group/host
# then, after verifying, prod:
ansible-playbook main.yaml --limit elasticsearch-prod
```

If the playbook reports `changed` on the apm_data task but you want to confirm the
restart fired, check `systemctl status elasticsearch` (uptime should be recent).

Single-node per env → expect a brief ingest gap during restart (Logstash will
reconnect automatically; it logged "Restored connection to ES" on the last
upgrade). For a multi-node cluster, restart one node at a time.

---

## Verify (after restart)

From any host that can reach ES (replace host per env):

```bash
for t in traces-apm metrics-apm logs-apm; do
  printf '%s -> ' "$t"
  curl -s -o /dev/null -w '%{http_code}\n' http://10.7.40.191:9200/_index_template/$t
done
# expect: 200 for each (was 404 before the change)

# or list them:
curl -s 'http://10.7.40.191:9200/_cat/templates/*apm*?v'
```

Then ping me — I'll bump **homelab-apm-server** to 8.18.1 +
`wait_for_integration:false`, deploy it to dev+prod, and confirm traces start
landing (the apm-server container in dev currently loops on the
"apm integration installed" precondition; it should clear within ~5s once these
templates exist and it's redeployed).

---

## Caveats

- **ILM references (elastic/elasticsearch#114591):** apm_data templates reference
  ILM policies; on 8.18 the plugin installs those policies too. If you see ILM
  "policy not found" warnings for `*-apm*` after enabling, flag them — usually
  benign but worth a look.
- **Naming mismatch (elastic/apm-server#13064):** apm_data installs
  `traces-apm@template` etc.; apm-server's legacy check expects `traces-apm`.
  Handled by `wait_for_integration: false` on apm-server — don't remove that.
- Keep `xpack.apm_data.enabled: true` set permanently (it's idempotent; templates
  are re-asserted on each ES start).
