# Exploring the Chart's Configuration

> Before changing a chart, find out what it lets you change. `helm show` reads a chart's defaults; `helm get` reads what a release actually used.

---

The WordPress you just installed runs entirely on defaults: the `user` login, a random password, a `LoadBalancer` Service. To change any of that you need two answers: **which settings exist**, and **which ones this release is using**. Helm splits them across two commands.

| Command | Reads from | Answers |
|---------|------------|---------|
| `helm show …` | the **chart** (repository or folder) | what can I configure, and what are the defaults? |
| `helm get …` | the **release** (Secrets in the cluster) | what did this install actually use? |

### The chart's defaults: `helm show values`

```bash
helm show values bitnami/wordpress --version 34.2.7 > wordpress-defaults.yaml
wc -l wordpress-defaults.yaml     # well over 1,000 lines
```

Don't read it top to bottom. Search for what you care about. Every setting is documented with an `@param` comment right above it:

```bash
grep -n '@param wordpressUsername' -A2 wordpress-defaults.yaml
```

```yaml
## @param wordpressUsername WordPress username
##
wordpressUsername: user
```

The settings you'll use in this part:

| Value | Default | What it controls |
|-------|---------|------------------|
| `wordpressUsername` | `user` | admin login |
| `wordpressPassword` | `""` (random) | admin password; empty means "generate one" |
| `wordpressBlogName` | `User's Blog!` | the site title |
| `replicaCount` | `1` | number of WordPress Pods |
| `service.type` | `LoadBalancer` | how the site is exposed |
| `persistence.size` | `10Gi` | size of the WordPress PVC |
| `ingress.enabled` | `false` | create an Ingress instead of exposing the Service directly |
| `resourcesPreset` | `micro` | CPU/memory requests and limits, as a named preset |
| `mariadb.enabled` | `true` | install the bundled MariaDB, or use an external database |
| `mariadb.auth.password` | `""` (random) | the database user's password |

> **Values for a dependency go under its name.** MariaDB is a separate chart, pulled in as a dependency. Its settings live under the `mariadb:` key, e.g. `mariadb.auth.password`, and are passed down to it. `helm show chart bitnami/wordpress` lists the dependencies.

The README is the friendlier version of the same list, with every parameter in a table:

```bash
helm show readme bitnami/wordpress --version 34.2.7 | less
```

It's also what the **Default values** and **README** tabs on [Artifact Hub](artifact-hub.md) show.

### What the release used: `helm get`

Now ask the release. First, the values you supplied:

```bash
helm get values my-wordpress
```

```text
USER-SUPPLIED VALUES:
null
```

`null`: you didn't override anything. Add `--all` to see the complete set the release was rendered with, defaults included:

```bash
helm get values my-wordpress --all | grep -E '^(wordpressUsername|replicaCount):'
```

```text
replicaCount: 1
wordpressUsername: user
```

The other `helm get` subcommands read the rest of what Helm stored for the release:

| Command | Shows |
|---------|-------|
| `helm get manifest my-wordpress` | the exact YAML Helm applied: every object, fully rendered |
| `helm get notes my-wordpress` | the NOTES printed after install |
| `helm get metadata my-wordpress` | chart name and version, app version, revision, status |
| `helm get hooks my-wordpress` | hook resources, if the chart has any |
| `helm get all my-wordpress` | all of the above in one dump |

`helm get manifest` is the one to remember, for two reasons.

**It shows where a value ends up.** A value's name tells you nothing about which object it lands in. Follow `wordpressUsername` into the rendered YAML:

```bash
helm get manifest my-wordpress | grep -A1 WORDPRESS_USERNAME
```

```yaml
            - name: WORDPRESS_USERNAME
              value: "user"
```

It became an environment variable on the WordPress Deployment, not a ConfigMap entry (the password, being secret, goes into the Secret `my-wordpress` instead). Every chart decides this differently, so when you want to know what a value really does, read the manifest:

```text
value (wordpressUsername)  →  template  →  object (Deployment env WORDPRESS_USERNAME)
```

**It's what you compare against the cluster.** The manifest is what Helm *thinks* it deployed. If someone changed an object with `kubectl edit`, the two no longer match ([drift](../helm-intro/benefits.md#%EF%B8%8F-state-is-stored-in-the-cluster)). `kubectl diff` shows the difference:

```bash
helm get manifest my-wordpress | kubectl diff -f -
```

No output means the cluster still matches what Helm deployed.

### Preview before you change

Once you know which value to change, render the result without touching the cluster:

```bash
helm template my-wordpress bitnami/wordpress --version 34.2.7 \
  --set service.type=NodePort --show-only templates/svc.yaml | grep -E 'kind|type:'
```

```text
kind: Service
  type: NodePort
```

`helm template` runs the same rendering as `helm install`, but only prints the YAML; `--show-only` limits it to one template file. It's the safest way to check what a value does before you apply it, and you'll use it again on the values pages.

---

[← Installing the WordPress Chart](installing-wordpress.md) · [↑ Contents](../../README.md) · [Uninstalling a Release →](uninstalling.md)
