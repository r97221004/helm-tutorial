# Benefits of Using Helm

> One chart for every environment, one command per lifecycle step, a history you can roll back to, and thousands of ready-made packages.

---

The [previous page](role-of-helm.md) listed what goes wrong with a folder of plain YAML. Each Helm feature answers one of those problems.

### 1. One chart, many environments

The same templates render differently for each environment. Only a small values file changes:

```text
my-app/                 # the chart, written once
values-dev.yaml         # replicaCount: 1, image.tag: latest
values-prod.yaml        # replicaCount: 3, image.tag: "1.4.2", ingress.enabled: true
```

```bash
helm install my-app ./my-app -f values-dev.yaml  -n dev
helm install my-app ./my-app -f values-prod.yaml -n prod
```

No copied folders, so no copies that slowly drift apart. A fix to the template reaches every environment at once.

### 2. The whole app is one unit

A release groups every object the chart created, so the full lifecycle is one command each:

| Step | Command |
|------|---------|
| Install | `helm install my-app ./my-app` |
| See what's installed | `helm list` |
| Change config or version | `helm upgrade my-app ./my-app --set image.tag=1.5.0` |
| Undo the last change | `helm rollback my-app` |
| Remove everything | `helm uninstall my-app` |

`helm uninstall` removes exactly what the release created, nothing more and nothing left behind.

### 3. Versioned history and rollback

Every install, upgrade and rollback creates a new **revision**, stored in the cluster with the values and rendered YAML it used:

```text
$ helm history my-app
REVISION  STATUS      CHART         APP VERSION  DESCRIPTION
1         superseded  my-app-0.1.0  1.4.1        Install complete
2         superseded  my-app-0.1.0  1.4.2        Upgrade complete
3         deployed    my-app-0.1.0  1.4.1        Rollback to 1
```

Rolling back doesn't need the old YAML in your repo: Helm already has it. A rollback is itself a new revision (3 above), so the history never loses anything.

### 4. Ready-made packages

You rarely write YAML for well-known software. Projects publish official charts, searchable on [Artifact Hub](https://artifacthub.io/):

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm install demo podinfo/podinfo --set replicaCount=2
```

The chart's maintainers have already worked out the Services, probes, RBAC and upgrade path. You only choose the values.

### 5. Dependencies

A chart can depend on other charts, listed in `Chart.yaml`:

```yaml
dependencies:
  - name: postgresql
    version: 1.x.x
    repository: https://charts.example.com   # any HTTP or oci:// chart repository
```

`helm dependency update` downloads them, and installing your chart installs the database with it, as one release.

### 6. Lifecycle hooks

A chart can mark a Job to run at a specific moment, for example a database migration **before** an upgrade, or a smoke test after an install (`helm test`). Plain `kubectl apply` has no notion of "before" or "after".

### 7. Sharing and versioning charts

A chart has its own version number, separate from the app's, and can be packaged into a `.tgz` and pushed to a repository or an OCI registry like any container image. Teams share internal charts the same way the community shares public ones.

### Summary

| Problem with plain YAML | Helm's answer |
|-------------------------|---------------|
| a copied folder per environment | one chart + one values file per environment |
| objects with no owner | a **release** that groups them |
| no record of what was deployed | **revisions** with values and rendered YAML |
| rollback means finding old files | `helm rollback` |
| writing YAML for common software | public charts on Artifact Hub |
| installing an app and its database separately | chart **dependencies** |
| no "run this before the upgrade" | **hooks** |

### The costs

Helm isn't free. Know the trade-offs before you reach for it:

- **Templates are harder to read than YAML.** Go templates with `{{- include ... | nindent 4 }}` take practice, and a large chart can be hard to follow. [Helm Fundamentals](../helm-fundamentals/overview.md) teaches you to read them.
- **Errors show up late.** A typo in a template only appears when the chart is rendered. `helm lint` and `helm template` catch most of them before you install.
- **State lives in the cluster.** The release history is stored in the cluster, not in Git. With a GitOps tool like Argo CD that role moves back to Git, which the [Argo CD tutorial](https://github.com/r97221004/argoCD-tutorial/blob/master/docs/argocd-helm/helm-charts.md) covers.
- **It can be too much for small things.** For one app with two environments, a lighter tool may do. That's the subject of the next page.

---

[← The Role of Helm in Kubernetes](role-of-helm.md) · [↑ Contents](../../README.md) · [Helm vs. Kustomize →](helm-vs-kustomize.md)
