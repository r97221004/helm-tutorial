# Setting Values via the CLI

> `--set key=value` overrides one value from the command line, on install or upgrade. Quick for experiments, easy to get wrong on upgrade, and gone from history the moment you close the terminal.

---

The [configuration page](exploring-configuration.md) found the values worth changing. The quickest way to change them is `--set`.

### Install with your own values

Start from a clean namespace, as the [cleanup page](cleaning-up.md) left it, and install WordPress with a fixed login, a site title, and a NodePort Service instead of the `LoadBalancer` that stays `<pending>`:

```bash
helm install my-wordpress bitnami/wordpress --version 34.2.7 \
  --set wordpressUsername=admin \
  --set wordpressPassword=change-me-123 \
  --set wordpressBlogName="My Helm Blog" \
  --set service.type=NodePort \
  --set service.nodePorts.http=30080 \
  --set mariadb.auth.rootPassword=root-change-me \
  --set mariadb.auth.password=db-change-me
```

Setting the database passwords also removes the [zombie volume](cleaning-up.md#the-zombie-volume-problem) problem: every reinstall now uses the same passwords as the data on disk.

Check what the release recorded:

```bash
helm get values my-wordpress
```

```text
USER-SUPPLIED VALUES:
mariadb:
  auth:
    password: db-change-me
    rootPassword: root-change-me
service:
  nodePorts:
    http: 30080
  type: NodePort
wordpressBlogName: My Helm Blog
wordpressPassword: change-me-123
wordpressUsername: admin
```

The site is now on port 30080 of the VM itself, no port-forward needed: `ssh -L 9090:localhost:30080 <YOUR_USER>@<YOUR_VM_IP>`, then open <http://localhost:9090> and log in as `admin`.

> **Passwords on the command line end up in your shell history**, and `helm get values` shows them to anyone who can read the release. Fine for a lab. For real installs, put them in a Kubernetes Secret and point the chart at it (`existingSecret`, `mariadb.auth.existingSecret`).

### `--set` syntax

| You want | Write | Result in values |
|----------|-------|------------------|
| a top-level value | `--set replicaCount=2` | `replicaCount: 2` |
| a nested value | `--set service.type=NodePort` | `service:` → `type: NodePort` |
| several at once | `--set replicaCount=2,service.type=NodePort` | both |
| a value with spaces | `--set wordpressBlogName="My Helm Blog"` | `wordpressBlogName: My Helm Blog` |
| a list item | `--set 'extraEnvVars[0].name=WORDPRESS_DEBUG'` | `extraEnvVars:` → `- name: WORDPRESS_DEBUG` |
| a key containing dots | `--set 'podAnnotations.prometheus\.io/scrape=true'` | `podAnnotations:` → `prometheus.io/scrape: true` |
| a **string**, never a number or bool | `--set-string 'extraEnvVars[0].value=true'` | `value: "true"` |
| the contents of a file | `--set-file 'customPostInitScripts.setup\.sh=./setup.sh'` | the file's text |
| a JSON object or list | `--set-json 'extraEnvVars=[{"name":"WORDPRESS_DEBUG","value":"true"}]'` | the parsed list |

Quote the whole argument when it contains `[`, `]` or `\`, so the shell leaves them alone.

### The type trap: use `--set-string`

`--set` guesses types: `true` becomes a boolean, `1234` a number. Kubernetes is strict about types, and some fields must be strings: environment variable values, annotations, labels. Render to see the difference:

```bash
helm template my-wordpress bitnami/wordpress --version 34.2.7 --show-only templates/deployment.yaml \
  --set 'extraEnvVars[0].name=WORDPRESS_DEBUG' --set 'extraEnvVars[0].value=true' | grep -A1 WORDPRESS_DEBUG
```

```yaml
            - name: WORDPRESS_DEBUG
              value: true          # a boolean: the API server rejects the Deployment
```

With `--set-string 'extraEnvVars[0].value=true'` the same line renders as `value: "true"`, which is what Kubernetes expects. When a value must be a string but looks like a number or a boolean, use `--set-string`.

### Upgrading: which values survive?

Change one thing on the running release, say two replicas:

```bash
helm upgrade my-wordpress bitnami/wordpress --version 34.2.7 --set replicaCount=2
```

Now look again:

```bash
helm get values my-wordpress
```

```text
USER-SUPPLIED VALUES:
replicaCount: 2
```

**Everything else is gone.** The visible damage: the Service is back to `LoadBalancer`, so port 30080 stops answering. The rest is reset too, just less visibly: the passwords are back to "generate one" (the chart happens to reuse the ones already in its Secrets), and `wordpressUsername` is back to `user` (WordPress keeps the `admin` account it created on first start, since the chart only applies these on first setup). This is Helm's most common surprise. On `helm upgrade`, Helm decides which values to start from:

| How you upgrade | Values used |
|-----------------|-------------|
| no `--set` or `-f` at all | the previous release's values, unchanged |
| any `--set` or `-f` | **chart defaults + only what you pass now**; previous values are dropped |
| `--reuse-values` + your changes | previous release's values + your changes |
| `--reset-values` | chart defaults only, ignoring the previous release |
| `--reset-then-reuse-values` + your changes | the **new chart's** defaults, then previous values, then your changes |

Put the original values back by repeating them, or keep them and add the change:

```bash
helm rollback my-wordpress 1                 # back to the values from the install
helm upgrade my-wordpress bitnami/wordpress --version 34.2.7 \
  --reuse-values --set replicaCount=2         # keep everything, change one thing
```

`--reuse-values` is handy, but it keeps the old *values* while you might be moving to a new *chart version*, whose defaults or keys may have changed. `--reset-then-reuse-values` handles that case better. Clearest of all is to stop depending on what the last command was: pass the full set of values every time, from a file. That's the [next page](values-files.md).

### See what changed

```bash
helm history my-wordpress
```

```text
REVISION  STATUS      CHART             APP VERSION  DESCRIPTION
1         superseded  wordpress-34.2.7  7.1.4        Install complete
2         superseded  wordpress-34.2.7  7.1.4        Upgrade complete
3         superseded  wordpress-34.2.7  7.1.4        Rollback to 1
4         deployed    wordpress-34.2.7  7.1.4        Upgrade complete
```

`helm get values my-wordpress --revision 2` shows the values of any revision, so you can always find out what a past upgrade did.

---

[← Cleaning Up Kubernetes Resources](cleaning-up.md) · [↑ Contents](../../README.md) · [Setting Values via Files →](values-files.md)
