# Uninstalling a Release

> `helm uninstall` deletes every object the release created and the release's history, in one command. "Every object the release created" is narrower than it sounds.

---

Removing an app installed with plain YAML means finding and deleting each object by hand. With Helm it's one command, because the release knows exactly what it created.

### Preview first

On anything that matters, check what a command will do before running it:

```bash
helm uninstall my-wordpress --dry-run
```

It prints the release that would be removed and deletes nothing.

### Uninstall

```bash
helm uninstall my-wordpress
```

```text
release "my-wordpress" uninstalled
```

Helm reads the release's stored manifest (the same YAML `helm get manifest` showed), deletes each object in it, then deletes the release's Secrets. Check the result:

```bash
helm list                  # empty: the release is gone
kubectl get pods           # Pods terminating, then gone
```

Like `helm install`, it returns once the API server accepts the deletions, not when every Pod has stopped. Add `--wait` to block until they're gone.

### Or: keep the history

A plain uninstall removes the release's history too, so the release name is free to reuse and `helm history` has nothing to show. The alternative is to delete the app but keep the record. You've already uninstalled, so this is for reference:

```bash
helm uninstall my-wordpress --keep-history
helm list --uninstalled      # my-wordpress  …  uninstalled
helm history my-wordpress    # every revision, the last one marked "uninstalled"
```

The objects are gone, but the revisions remain, so you can see what was deployed and even bring it back with `helm rollback my-wordpress <revision>`. While the history exists, the name stays taken: a new `helm install my-wordpress` fails until you run a plain `helm uninstall my-wordpress` to clear it.

| | `helm uninstall` | `helm uninstall --keep-history` |
|---|---|---|
| App objects | deleted | deleted |
| Release history | deleted | kept, status `uninstalled` |
| `helm list` | gone | only with `--uninstalled` |
| Reuse the release name | yes | no, until the history is removed |
| `helm rollback` | not possible | possible |

### What uninstall does *not* delete

Helm deletes only the objects in the release's manifest. Some objects related to the app were never in it:

| Left behind | Why |
|-------------|-----|
| **PVCs created by a StatefulSet** | Created by the StatefulSet controller from `volumeClaimTemplates`, not by Helm. Kubernetes keeps them by default, so the data survives the StatefulSet. |
| **Objects with `helm.sh/resource-policy: keep`** | The chart asked Helm to leave them, usually for data that's expensive to lose. |
| **CRDs from a chart's `crds/` folder** | Helm installs them but never deletes them, since other releases may use them. |
| **Anything created at runtime** | e.g. Jobs, Secrets or ConfigMaps created by an operator or by the app itself. |
| **The namespace** | Helm didn't create it (unless you used `--create-namespace`, and even then it's not deleted). |

WordPress hits the first row. Look:

```bash
kubectl get pvc
```

```text
NAME                          STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS
data-my-wordpress-mariadb-0   Bound    pvc-3f9c…       8Gi        RWO            local-path
```

The WordPress PVC (`my-wordpress`) was in the manifest and is gone. The MariaDB PVC wasn't, so it's still here, along with the whole database. The [next page](cleaning-up.md) explains why that matters and how to clean it up.

---

[← Exploring the Chart's Configuration](exploring-configuration.md) · [↑ Contents](../../README.md) · [Cleaning Up Kubernetes Resources →](cleaning-up.md)
