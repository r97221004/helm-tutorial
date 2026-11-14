# Helm in Practice

<p align="center">
  <img src="docs/images/helm-logo.svg" width="120" alt="Helm logo" />
</p>

> _Learn Helm by installing real charts and building your own, one concept at a time._

![Helm](https://img.shields.io/badge/Helm-4-0F1689?logo=helm&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.30%2B-326CE5?logo=kubernetes&logoColor=white)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)

> **Problem**: _TODO_
>
> **Solution**: _TODO_

## What You'll Learn

_TODO_

## Prerequisites

_TODO_ — 建議先完成 [Kubernetes in Practice](https://github.com/r97221004/k8s-tutorial)。

## Table of Contents

**Part 1 — What is Helm?**

- [**The Role of Helm in Kubernetes**](docs/helm-intro/role-of-helm.md) — Kubernetes has no idea which objects form one app; Helm packages them as charts and tracks installs as releases.
- [**Benefits and Limitations of Helm**](docs/helm-intro/benefits.md) — seven benefits and five limitations: simpler deployments, consistent environments and rollbacks, weighed against templating complexity, chart security, in-cluster state and risky upgrades.
- [**Helm vs. Kustomize**](docs/helm-intro/helm-vs-kustomize.md) — templates and values vs. base and patches: the same change both ways, and when to use which.
- [**Helm Architecture**](docs/helm-intro/architecture.md) — how developers, chart repositories and CI/CD fit together; the chart files and common commands; and what `helm install` does, from rendering to the release Secret.

**Part 2 — Installing Tools**

- [**Installing Tools**](docs/installing-tools/overview.md) — _TODO_

**Part 3 — Helm Fundamentals**

- [**Helm Fundamentals**](docs/helm-fundamentals/overview.md) — _TODO_

**Part 4 — Creating Our Own Helm Charts**

- [**Creating Our Own Helm Charts**](docs/creating-charts/overview.md) — _TODO_
