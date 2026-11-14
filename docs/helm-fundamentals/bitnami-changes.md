# Changes to Bitnami Charts and Images

> The WordPress chart in this part comes from Bitnami, which changed what it gives away for free in 2025. Charts are still free; free images now come only as `latest`.

---

For years, Bitnami's free charts and images were the default way to install common software with Helm, and most Helm courses, including the one this tutorial follows, use its WordPress chart. In 2025 Bitnami (now part of Broadcom) changed that model. You'll still use the Bitnami WordPress chart in this part, but you should know what changed and why it matters.

### What changed on August 28, 2025

| Before | After |
|--------|-------|
| Every image version free on `docker.io/bitnami`, e.g. `bitnami/wordpress:6.8.2-debian-12-r5` | Only the **`latest`** tag of a limited set of hardened images stays free on `docker.io/bitnami` (also published under `docker.io/bitnamisecure`) |
| Versioned tags, Debian-based | Old versioned tags moved to **`docker.io/bitnamilegacy`**, an archive that gets **no updates** |
| Charts and images both free for production | Pinned versions, long-term support branches and vendor support come with the paid **Bitnami Secure Images** subscription |

The **charts** are still published and maintained. The source of both charts and images stays on GitHub under the Apache 2.0 license. You can check the full announcement in [bitnami/containers#83267](https://github.com/bitnami/containers/issues/83267).

### What it means for the WordPress chart

Current chart versions point at the free `latest` images, so a plain install works. The chart's `values.yaml` says:

```yaml
image:
  registry: registry-1.docker.io
  repository: bitnami/wordpress
  tag: latest
```

That's fine for learning. But for anything real, `latest` is a problem:

- **You can't pin the app version.** `latest` points at whatever Bitnami built most recently, so two installs a week apart can run different WordPress versions, and a Pod that lands on a node without the image cached pulls whatever `latest` is that day.
- **Rollback is weaker.** `helm rollback` restores the chart's YAML, but if that YAML says `tag: latest`, the image you roll back to is still today's `latest`.

That breaks the "versioned packages" benefit from [Benefits and Limitations](../helm-intro/benefits.md#-versioned-packages), and it's a concrete example of the [security limitation](../helm-intro/benefits.md): a source you depend on can change its terms.

### If an older install stops pulling images

Charts released **before** the change reference versioned tags such as `bitnami/wordpress:6.8.2-debian-12-r5`, which no longer exist on `docker.io/bitnami`. Pods from those charts fail with `ImagePullBackOff`. Bitnami's suggested temporary workaround is to point each image at the legacy archive:

```bash
helm upgrade my-wordpress bitnami/wordpress --version <same-chart-version> \
  --reuse-values \
  --set image.repository=bitnamilegacy/wordpress \
  --set mariadb.image.repository=bitnamilegacy/mariadb \
  --set global.security.allowInsecureImages=true
```

`allowInsecureImages` is needed because recent Bitnami charts refuse to render when an image is swapped for one they don't recognize. Legacy images get no security fixes, so treat this as a bridge while you migrate, not a fix.

### What this tutorial does

- **Part 3 uses the current Bitnami WordPress chart with its default `latest` images.** It's still the most common example in Helm material, and it exercises everything this part teaches: repositories, a dependency (MariaDB), Secrets, PVCs and many values.
- **For your own work, prefer charts whose images you can pin**: the project's official chart, a maintained alternative found on [Artifact Hub](artifact-hub.md), or your own chart, which you'll build in [Part 4](../creating-charts/overview.md).

---

[← Installing Tools](../installing-tools/install-tools.md) · [↑ Contents](../../README.md) · [Exploring Artifact Hub →](artifact-hub.md)
