# Cleaning Up Kubernetes Resources

> A leftover PVC isn't just wasted disk. Reinstall the same release on top of it and the new database password won't match the old data, and WordPress can't connect.

---

The [previous page](uninstalling.md) left the MariaDB PVC behind. That's Kubernetes protecting your data, but a forgotten volume can surprise you later. This page shows the surprise, then cleans up properly.

### The "zombie volume" problem

Reinstall WordPress with the same release name, as if starting fresh:

```bash
helm install my-wordpress bitnami/wordpress --version 34.2.7
```

Here's what happens:

| Step | Password Secret | MariaDB data |
|------|-----------------|--------------|
| First install | the chart generates password **A** and stores it in Secret `my-wordpress-mariadb` | initializes an empty database with password **A**, on PVC `data-my-wordpress-mariadb-0` |
| `helm uninstall` | Secret deleted | StatefulSet deleted, **PVC kept**, still holding a database with password **A** |
| Reinstall | no Secret to reuse, so a new random password **B** | the new StatefulSet finds the **old** PVC by name and reuses it. The database is already initialized, so it keeps password **A** |
| Result | WordPress connects with **B** | the database expects **A** → `Access denied` |

WordPress can't reach its database and keeps restarting:

```bash
kubectl get pods
```

```text
NAME                            READY   STATUS             RESTARTS      AGE
my-wordpress-6c7d9f8b5-q8n2m    0/1     CrashLoopBackOff   4 (30s ago)   4m
my-wordpress-mariadb-0          1/1     Running            0             4m
```

```bash
kubectl logs deploy/my-wordpress | grep -i denied
```

The PVC name is built from the release name, so reusing the name reuses the volume. The old data comes back from the dead, with credentials nobody has anymore.

> **Two ways to avoid it:** remove the leftover PVC before reinstalling (below), or set the passwords yourself (`mariadb.auth.password`, `mariadb.auth.rootPassword`) so they're the same on every install. You'll set values on the [next page](values-cli.md).

Uninstall again before cleaning up:

```bash
helm uninstall my-wordpress
```

### Find what's left

Every object from the chart carries the release's instance label, including PVCs made from the StatefulSet's template. Search for anything still labeled:

```bash
kubectl get all,pvc,secret,configmap -l app.kubernetes.io/instance=my-wordpress
```

Labels only find objects that have them. To see everything in the namespace:

```bash
kubectl get all,pvc,secret,configmap
```

### Delete the leftovers

```bash
kubectl delete pvc data-my-wordpress-mariadb-0
```

With local-path, the StorageClass's **reclaim policy** is `Delete`, so removing the PVC also removes its PersistentVolume and the data directory on the node:

```bash
kubectl get pv      # the volume is gone too
```

> With a `Retain` reclaim policy, common for production storage, the PV and its data survive the PVC. The PV shows `Released` and has to be deleted, and its disk cleaned, by hand. Check `kubectl get storageclass` before assuming data is gone, or that it's safe.

### When you're done: delete the namespace

Deleting the namespace removes everything still inside it, PVCs included. **Skip this for now if you're continuing**: the values pages install WordPress again in `wordpress`. When you've finished the lab:

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace wordpress
```

Switch your context back first, or the next `kubectl` and `helm` commands will run against a namespace that no longer exists.

If you deleted it already, recreate it before the next page:

```bash
kubectl create namespace wordpress
kubectl config set-context --current --namespace=wordpress
```

### A cleanup checklist

After uninstalling any chart that stores data:

1. **`helm list -A`** — is the release really gone, or kept with `--keep-history`?
2. **`kubectl get pvc`** — any volumes from StatefulSets left behind?
3. **`kubectl get pv`** — any `Released` volumes still holding data?
4. **`kubectl get all,secret,configmap`** — anything created at runtime, or kept by `helm.sh/resource-policy: keep`?
5. **`kubectl get crd`** — any CRDs the chart installed that no other release needs?

Or put each lab in its own namespace and delete the namespace when you're done, as this part does.

---

[← Uninstalling a Release](uninstalling.md) · [↑ Contents](../../README.md) · [Setting Values via the CLI →](values-cli.md)
