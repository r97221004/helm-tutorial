# Exploring Artifact Hub

> Artifact Hub is the search engine for Helm charts. It doesn't host charts; it indexes thousands of repositories and tells you which chart to trust, what it installs, and how to configure it.

---

Before you install a public chart, you have to find it, and usually choose between several charts for the same software. [Artifact Hub](https://artifacthub.io/) is the place to do that. It's a CNCF project that indexes Helm charts (and other Kubernetes packages) from thousands of repositories.

> **Artifact Hub doesn't store charts.** It points at the repository that does. You install from that repository, as the [next page](managing-repositories.md) shows, not from Artifact Hub.

### Searching

Search for `wordpress` on [artifacthub.io](https://artifacthub.io/) and filter **Kind → Helm charts**. You'll get dozens of results. Narrow them with the filters on the left:

| Filter | What it tells you |
|--------|-------------------|
| **Official** | published by the owner of the software itself, e.g. the Prometheus chart from the Prometheus community |
| **Verified publisher** | the repository's owner has proved they control it |
| **CNCF** | the chart belongs to a CNCF project |
| **Signed** | the chart is cryptographically signed, so you can verify it wasn't tampered with |

For WordPress, pick **`bitnami/wordpress`** from the Bitnami repository: the chart this part uses.

You can run the same search from the terminal. `helm search hub` queries Artifact Hub directly, no repository setup needed:

```bash
helm search hub wordpress --list-repo-url
```

```text
URL                                         CHART VERSION  APP VERSION  DESCRIPTION                     REPO URL
https://artifacthub.io/packages/helm/...    34.2.7         7.1.4        WordPress is the world's ...    https://charts.bitnami.com/bitnami
…
```

Results come back in Artifact Hub's ranking, so the Bitnami chart may not be first; look for the row whose **REPO URL** is `https://charts.bitnami.com/bitnami`. That column, added by `--list-repo-url`, is what you actually need: the repository to add before installing.

### Reading a chart's page

Open the `bitnami/wordpress` package page. Each part answers a question you should ask before installing:

| Part of the page | Question it answers |
|------------------|---------------------|
| **Install** button | Which repository URL and commands do I need? |
| **Versions** list | How often is it released, and is it still maintained? |
| **Chart version / App version** | Which chart release, and which WordPress inside it? (the two versions from [Benefits](../helm-intro/benefits.md#-versioned-packages)) |
| **README** | What does the chart do, and what are its main options? |
| **Default values** | Every setting you can change, with its default. The same as `helm show values`. |
| **Templates** | Which Kubernetes objects does it create? |
| **Changelog** | What changed between versions, and what might break on upgrade? |
| **Security report** | Do the chart's images have known vulnerabilities? |

The **Default values** view is the one you'll use most. For WordPress it shows settings such as `wordpressUsername`, `service.type` and `persistence.size`, which you'll change in [Setting Values via the CLI](values-cli.md).

### Choosing between charts

When several charts install the same software, prefer one that is:

1. **Official, or from a well-known publisher.** It's maintained by people who know the software.
2. **Recently updated.** Look at the date of the latest version. A chart untouched for a year is probably out of date.
3. **Documented.** A README that explains values, and a values schema, save you from reading templates.
4. **Clean on the security report.** Fewer known vulnerabilities in its images.

Then read what it creates before installing it, as [Benefits and Limitations](../helm-intro/benefits.md) recommends: render it with `helm template`, or browse the **Templates** tab.

---

[← Changes to Bitnami Charts and Images](bitnami-changes.md) · [↑ Contents](../../README.md) · [Managing Helm Repositories →](managing-repositories.md)
