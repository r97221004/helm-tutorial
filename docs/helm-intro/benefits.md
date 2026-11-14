# Benefits and Limitations of Helm

> One command per lifecycle step, one chart for every environment, and a large catalog of ready-made packages, paid for with a template language, state hidden in the cluster, and upgrades that need care.

---

The [previous page](role-of-helm.md) showed what Helm fixes for a single app. This page weighs it as a tool: what you gain by managing Kubernetes resources with Helm, and what it costs.

### At a glance

| ✅ Benefit | What it means |
|-----------|---------------|
| **Simpler deployments** | install, upgrade, roll back or remove a whole app with one command |
| **Consistent environments** | one chart, a small values file per environment |
| **Release management** | every change is a revision you can roll back to |
| **Collaboration** | thousands of public charts, and private repositories for your own |
| **Versioned packages** | a chart has a version you can pin, like any dependency |
| **Templating flexibility** | conditionals and loops that plain YAML and Kustomize don't have |
| **Hooks and tests** | run a migration before an upgrade, and tests after an install |

| ⚠️ Limitation | What it means |
|---------------|---------------|
| **Learning curve** | Go templates and chart structure take time to learn |
| **Over-templating** | charts can become harder to maintain than the YAML they replaced |
| **Security** | a community chart runs someone else's YAML in your cluster |
| **State in the cluster** | Helm's record can disagree with what's actually running |
| **Upgrades need care** | a careless version bump or renamed value can break them |

The examples below reuse `app-chart` and the `my-app` release from the previous page.

### ✅ Simpler deployments

A release groups every object the chart created, so each lifecycle step is one command, however many objects the app has:

| Step | Command |
|------|---------|
| Install | `helm install my-app ./app-chart -n dev` |
| See what's installed | `helm list -n dev` |
| Change config or version | `helm upgrade my-app ./app-chart -n dev --set web.tag=1.5.0` |
| Undo the last change | `helm rollback my-app -n dev` |
| Remove everything | `helm uninstall my-app -n dev` |

`helm upgrade` sends the new version of all eleven objects, and Kubernetes changes only what differs; `helm uninstall` removes exactly what the release created.

### ✅ Consistent environments

Every environment uses the same templates. Only a small values file differs:

```text
app-chart/              # the chart, written once
values-dev.yaml         # web.replicaCount: 1, web.tag: "1.5.0" (trying the next release)
values-prod.yaml        # web.replicaCount: 3, ingress.enabled: true (tag from appVersion: 1.4.2)
```

```bash
helm install my-app ./app-chart -n dev  -f values-dev.yaml
helm install my-app ./app-chart -n prod -f values-prod.yaml
```

A fix to a template reaches every environment at once, and the values files show at a glance exactly how prod differs from dev.

### ✅ Release management

Every install, upgrade and rollback creates a new **revision**, stored in the cluster with the values and rendered YAML it used:

```text
$ helm history my-app -n dev
REVISION  STATUS      CHART            APP VERSION  DESCRIPTION
1         superseded  app-chart-2.1.0  1.4.2        Install complete
2         superseded  app-chart-2.2.0  1.5.0        Upgrade complete
3         deployed    app-chart-2.1.0  1.4.2        Rollback to 1
```

Rolling back doesn't need the old YAML in your repo: Helm already has it. A rollback is itself a new revision (3 above), so the history never loses anything.

### ✅ Collaboration

You rarely write YAML for well-known software. Projects publish official charts, searchable on [Artifact Hub](https://artifacthub.io/):

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm install demo podinfo/podinfo --set replicaCount=2
```

- **Public charts.** The maintainers have already worked out the Services, probes, RBAC and upgrade path; you only choose the values.
- **Private charts.** A platform team publishes a standard chart to a private repository or OCI registry, and every service uses it, so every service gets the same labels, probes and security settings.
- **Documented settings.** A chart's `values.yaml`, README and optional `values.schema.json` describe what each setting does, which makes it a reusable component rather than a folder to copy.

### ✅ Versioned packages

A chart carries two versions in `Chart.yaml`:

```yaml
version: 2.1.0       # the chart: bump it on any change to the chart, appVersion included
appVersion: "1.4.2"  # the app it deploys, shown in helm list
```

You can pin a chart version the way you pin a library, and a chart can depend on other charts at pinned versions:

```bash
helm install demo podinfo/podinfo --version 6.7.1   # this exact chart, every time
```

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: 1.x.x
    repository: https://charts.example.com   # any HTTP or oci:// chart repository
```

`helm dependency update` downloads them, and installing your chart installs the database with it, as one release.

### ✅ Templating flexibility

Templates can do what plain YAML and Kustomize can't: make decisions and repeat things.

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# … created only when the values ask for it
{{- end }}
```

Loops (`range`) generate one object per item in a list, and functions handle defaults, quoting and indentation. One chart can then cover cases that would otherwise need separate sets of YAML.

### ✅ Hooks and tests

A chart can mark a Job to run at a specific moment, for example a database migration **before** an upgrade. It can also ship test Pods that `helm test my-app` runs after an install, which makes charts easy to check in CI. Plain `kubectl apply` has no notion of "before", "after" or "is it working".

### ⚠️ Learning curve

Helm adds its own concepts on top of Kubernetes: chart structure, values precedence, releases. And its templates are written in Go's template language, which is powerful but not always obvious:

```yaml
  labels:
    {{- include "app-chart.labels" . | nindent 4 }}
```

The `-` trims whitespace, `.` is the current scope, and `nindent 4` fixes the YAML indentation; get one wrong and the output is broken YAML. Errors also show up late, only when the chart is rendered. `helm lint` and `helm template` catch most of them before you install, and [Helm Fundamentals](../helm-fundamentals/overview.md) teaches you to read charts like this.

### ⚠️ Over-templating

It's tempting to turn every field into a value "just in case". The result is a chart where every line is `{{ .Values.something }}`, the `values.yaml` is as long as the YAML it replaced, and nobody can tell what the chart actually produces without rendering it.

A rule of thumb: **template what actually differs between installs, and hard-code the rest.** And for a small app, Helm itself may be more than you need; see [Helm vs. Kustomize](helm-vs-kustomize.md).

### ⚠️ Security of community charts

Installing a public chart means running someone else's YAML in your cluster, often with ClusterRoles, privileged Pods or images from registries you don't control. Before installing one:

- **Render it first** with `helm template` and read what it creates, especially RBAC.
- **Check who maintains it** and how recently it was updated. Prefer the project's official chart.
- **Pin the chart version**, so a new release can't change what you deploy without you noticing.
- **Plan for the source to change.** In August 2025, for example, Bitnami stopped publishing most of its free container images and moved the existing ones to a legacy repository, so many teams had to rework charts they'd relied on for years.

### ⚠️ State is stored in the cluster

Helm's record of each release lives in Secrets in the release's namespace, not in Git. Delete them and Helm forgets the release, as the [architecture page](architecture.md#where-releases-are-stored) explains. The bigger day-to-day issue is **drift**:

- Someone runs `kubectl edit` on an object Helm deployed. The live object no longer matches what Helm thinks it deployed, and `helm get manifest` still shows the old version.
- Helm only looks at the cluster when you run a command, so nothing notices.
- What the next upgrade does with the manual change depends on the Helm version. Helm 3 does a three-way merge and overwrites fields the chart sets. Helm 4 uses server-side apply for new installs, where Kubernetes tracks which tool owns each field, so the change may be overwritten or reported as a conflict.

Either way, don't rely on it: change the app through the chart and `helm upgrade`, never `kubectl edit`. Continuous drift detection is what GitOps tools like Argo CD add on top, covered in the [Argo CD tutorial](https://github.com/r97221004/argoCD-tutorial/blob/master/docs/argocd-helm/helm-charts.md).

### ⚠️ Upgrades need care

`helm upgrade` is one command, but it's only as safe as the chart change behind it:

- **Version numbers are only a promise.** A chart is supposed to follow SemVer, but if a breaking change ships as a minor bump (2.1 → 2.2), anyone who trusted the number gets a surprise.
- **Renamed values are silently ignored.** If a new chart version renames `web.tag` to `web.image.tag`, your old values file still "works": Helm ignores the unknown key and deploys the default. A `values.schema.json` that sets `additionalProperties: false` makes Helm reject unknown keys instead.
- **Some changes can't be applied in place.** Fields like a Deployment's `selector` are immutable, so an upgrade that changes them fails.
- **CRDs aren't upgraded.** CRDs in a chart's `crds/` folder are installed once and never upgraded or deleted by Helm.
- **APP VERSION can lie.** The APP VERSION column in `helm list` and `helm history` comes from `Chart.yaml`, not from the image that's actually running. After `helm upgrade my-app ./app-chart --set web.tag=1.5.0`, it still says 1.4.2. To see the real image, check the Deployment: `kubectl get deploy my-app-web -o jsonpath='{..image}'`.

Read the chart's changelog, render the new version with `helm template` and compare, and try the upgrade in dev before prod.

### When Helm is worth it

Helm pays off when you install third-party software, deploy the same app to several environments, or share an app with other teams. For a single small app, weigh the limitations above first.

---

[← The Role of Helm in Kubernetes](role-of-helm.md) · [↑ Contents](../../README.md) · [Helm vs. Kustomize →](helm-vs-kustomize.md)
