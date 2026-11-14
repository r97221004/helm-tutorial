# Setting Values via Files

> Put your values in a YAML file, pass it with `-f`, and keep it in Git. Every upgrade then uses the full, reviewed set of values, not whatever the last command line happened to contain.

---

The [previous page](values-cli.md) ended with the main weakness of `--set`: on upgrade, values you don't repeat are dropped, and nothing records what you typed. A values file fixes both.

### Write a values file

A values file uses the same structure as the chart's `values.yaml`, but contains **only what you change**. This repository has one for WordPress, [`examples/helm-fundamentals/wordpress.yaml`](../../examples/helm-fundamentals/wordpress.yaml):

```yaml
wordpressUsername: admin
wordpressPassword: change-me-123
wordpressEmail: admin@example.com
wordpressBlogName: My Helm Blog

service:
  type: NodePort
  nodePorts:
    http: 30080

persistence:
  size: 2Gi

mariadb:
  auth:
    rootPassword: root-change-me
    password: db-change-me
  primary:
    persistence:
      size: 2Gi
```

Compare it with the `--set` install from the previous page: the same settings, but readable, commented, and saved.

> Don't start from a copy of `helm show values`. Copying all 1,000+ lines makes it impossible to see what you actually changed, and freezes the chart's defaults into your file so you miss improvements in later versions. Copy in only the keys you change.

### Install and upgrade with `-f`

Clone this repository on the VM to get the file:

```bash
git clone https://github.com/r97221004/helm-tutorial.git
cd helm-tutorial
```

Remove the release from the previous page and install from the file. Its passwords match the ones from the previous page, so the existing database volume works with them:

```bash
helm uninstall my-wordpress
helm install my-wordpress bitnami/wordpress --version 34.2.7 \
  -f examples/helm-fundamentals/wordpress.yaml
```

`helm get values my-wordpress` now shows exactly the file's contents. Later changes are edits to the file, applied with the same command every time:

```bash
helm upgrade my-wordpress bitnami/wordpress --version 34.2.7 \
  -f examples/helm-fundamentals/wordpress.yaml
```

Because the file always carries the full set of your values, the "everything else is gone" problem from the previous page can't happen. The file *is* the complete list.

> **`helm upgrade --install`** installs the release if it doesn't exist and upgrades it if it does. With a values file, it's the one command CI pipelines run for every deploy.

### Layer several files

Pass `-f` more than once to combine files. Later files override earlier ones, key by key. [`wordpress-scale.yaml`](../../examples/helm-fundamentals/wordpress-scale.yaml) adds only what's needed to scale up:

```yaml
replicaCount: 2
resourcesPreset: small
```

```bash
helm upgrade my-wordpress bitnami/wordpress --version 34.2.7 \
  -f examples/helm-fundamentals/wordpress.yaml \
  -f examples/helm-fundamentals/wordpress-scale.yaml
```

This is how one chart serves several environments, the pattern from [The Role of Helm](../helm-intro/role-of-helm.md): a shared file with what every environment has in common, plus a small file per environment.

```text
values/
├── common.yaml     # login, blog name, database settings
├── dev.yaml        # replicaCount: 1, small disks
└── prod.yaml       # replicaCount: 3, ingress.enabled: true, bigger disks
```

```bash
helm upgrade --install my-wordpress bitnami/wordpress --version 34.2.7 -f values/common.yaml -f values/prod.yaml
```

### Which value wins?

When the same key is set in several places, Helm merges them in this order, lowest to highest:

| Precedence | Source |
|------------|--------|
| 1 (lowest) | the chart's `values.yaml` (and a dependency's own defaults) |
| 2 | `-f` files, in the order given; the last one wins |
| 3 (highest) | `--set`, `--set-string`, `--set-json`, `--set-file` |

`--set` beats every file, wherever it appears on the command line. Preview the result to be sure:

```bash
helm template my-wordpress bitnami/wordpress --version 34.2.7 --show-only templates/deployment.yaml \
  -f examples/helm-fundamentals/wordpress.yaml \
  -f examples/helm-fundamentals/wordpress-scale.yaml \
  --set replicaCount=3 | grep 'replicas:'
```

```text
  replicas: 3
```

Unset in `wordpress.yaml`, 2 in `wordpress-scale.yaml`, 3 from `--set`: the `--set` wins. Use `--set` for a one-off override on top of your files, like a test image tag, and keep everything permanent in the files.

### Keep values files in Git

A values file in Git gives you what `--set` never does:

- **Review.** A change to production is a pull request someone can read.
- **History.** `git log` shows who changed which value, when, and why.
- **Reproducibility.** The same chart version plus the same file gives the same release, on any cluster.

The one thing that doesn't belong in Git is a plain-text password, like the lab passwords in `wordpress.yaml`. For real installs, keep secrets in a Kubernetes Secret and reference it with `existingSecret`, or use a tool such as Sealed Secrets, SOPS or External Secrets.

### Clean up

When you're done with Part 3:

```bash
helm uninstall my-wordpress
kubectl delete pvc data-my-wordpress-mariadb-0
kubectl config set-context --current --namespace=default
kubectl delete namespace wordpress
```

The [cleanup page](cleaning-up.md#a-cleanup-checklist) has the full checklist.

---

[← Setting Values via the CLI](values-cli.md) · [↑ Contents](../../README.md) · [Creating Our Own Helm Charts →](../creating-charts/overview.md)
