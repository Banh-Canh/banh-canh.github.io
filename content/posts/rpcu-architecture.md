---
title: 'Building RPCU: An Open Source Cloud Platform from Scratch'
date: 2026-06-14
tags: ['infrastructure', 'openstack', 'kubernetes', 'nixos', 'gitops', 'sre']
description: 'A look at the architecture, design decisions, and technology choices behind RPCU — an open source cloud infrastructure platform built on OpenStack, Kubernetes, and NixOS.'
---

## What is RPCU?

<img src="/images/rpcu-logo.png" alt="RPCU" width="150" style="float: right; margin: 0 0 1em 1.5em;">

RPCU is an open source infrastructure platform built on OpenStack. The goal is straightforward: provide a multi-tenant cloud that an organization can deploy, operate, and fully understand - without vendor lock-in, without hidden abstractions, and with everything described as code.

RPCU is a lab and R&D project running on three baremetal servers from Hetzner. It's not production-hardened for thousands of users, and it's not trying to compete with managed cloud offerings. It's a place to learn, experiment, and build something that works end to end.

{{< notice note "The Constraint" >}}
This runs on rented hardware with a virtual switch. Hetzner provides vSwitch connectivity between baremetal servers, but vSwitches come with limitations — no multicast, a 1400 MTU cap on the VLAN, and no physical network gear to lean on. Achieving high availability on top of that, without dedicated load balancers or physical redundancy, is the real engineering problem.
{{< /notice >}}

## The Problem

OpenStack is powerful but complex. Kubernetes is flexible but has its own operational weight. NixOS is reproducible but has a steep learning curve. Getting them to work together - cleanly, declaratively, and without click-ops - is a real engineering challenge.

RPCU is an attempt to solve that integration problem. Not by building another abstraction layer, but by choosing the right tools and wiring them together with Infrastructure as Code.

## The Three Pillars

RPCU is organized into three repositories:

### [Aletheia](https://github.com/RPCU/aletheia) — The Documentation

VitePress documentation site. Architecture, install guides, operational procedures. In an IaC project, docs are part of the product since if it's not documented, it's not reproducible.

### [Argus](https://github.com/RPCU/argus) — The GitOps Engine

Flux CD repository that manages the entire stack. Clusters, CNI, storage, OpenStack control plane are all reconciled from Git. **A change is a Git commit.** No click-ops.

### [Hephaestus](https://github.com/RPCU/hephaestus) — The Operating System

NixOS modules deployed via Colmena. Every machine is identical, reproducible, and version-locked.

## Architecture Overview

RPCU runs a multi-cluster architecture. The key thing to understand: **the OpenStack cluster is the foundation**. It runs on baremetal hardware and provides the VM infrastructure that the management cluster runs on.

```
┌─────────────────────────────────────────────────────────────────┐
│                  OpenStack Cluster (baremetal)                  │
│  lucy                  makise                  quinn            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                 Yaook OpenStack Operators                 │  │
│  │  ┌────────┐ ┌────────┐ ┌──────────┐ ┌───────────────┐     │  │
│  │  │ Nova   │ │Neutron │ │ Keystone │ │   Glance      │     │  │
│  │  │(compute│ │(OVN)   │ │(identity)│ │  (images)     │     │  │
│  │  └────────┘ └────────┘ └──────────┘ └───────────────┘     │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌────────┐ ┌─────────┐ ┌────────┐ ┌────────┐ ┌────────────┐    │
│  │Cilium  │ │ Rook/   │ │cert-   │ │kgateway│ │Crossplane  │    │
│  │(CNI)   │ │ Ceph    │ │manager │ │(GW API)│ │+ Zitadel   │    │
│  └────────┘ └─────────┘ └────────┘ └────────┘ └────────────┘    │
│                                │                                │
│              CAPO provisions VMs for mgmt cluster               │
└────────────────────────────────┼────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│               Management Cluster (OpenStack VMs)                │
│  ┌────────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Cluster API│  │  Sveltos  │  │Crossplane │  │  Cilium   │    │
│  │ (CAPI/CAPO)│  │           │  │(chihiro)  │  │(no L2 LB) │    │
│  └─────┬──────┘  └─────┬─────┘  └───────────┘  └───────────┘    │
│        │              │                                         │
└────────┼──────────────┼─────────────────────────────────────────┘
         │              │
         ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Workload Clusters (CAPI)                     │
│  CAPI-provisioned OpenStack VMs, managed via Sveltos            │
│  Opt-in add-ons: Cilium, Flux, OIDC RBAC                        │
└─────────────────────────────────────────────────────────────────┘
```

**OpenStack Cluster**: The foundation. Runs on three baremetal NixOS nodes (lucy, makise, quinn) with kubeadm and kube-vip HA. The full OpenStack control plane (Keystone, Nova, Neutron/OVN, Glance, Cinder, Octavia, Designate, Barbican, Horizon) runs as Kubernetes operators via Yaook. Rook/Ceph provides distributed storage. This cluster also manages the shared Zitadel identity platform via Crossplane.

{{< notice info "Not Managed by CAPI" >}}
The OpenStack cluster is **not** managed by Cluster API. It was bootstrapped manually with kubeadm. Only the management and workload clusters use CAPI.
{{< /notice >}}

**Management Cluster**: Runs as CAPO-provisioned OpenStack VMs. Bootstrapped with kind + `clusterctl`, intended to self-manage after `clusterctl move`. Runs Cluster API providers (to provision workload clusters), Sveltos (for multi-cluster add-on management), and a small Crossplane install (only the chihiro OIDC client). Uses OpenStack CCM + Octavia for LoadBalancer Services and Cinder CSI for storage, **not** Cilium L2 / Rook like the OpenStack cluster.

**Workload Clusters**: CAPI-provisioned OpenStack VMs. Managed declaratively by Sveltos from the management cluster via opt-in `ClusterProfile` resources gated by labels. Add-ons (Cilium, Flux, OIDC RBAC) are pushed per-cluster, not forced onto every cluster.

## Technology Choices

### [NixOS](https://github.com/NixOS/nixos)

Every node (baremetal and VM) runs the same NixOS configuration. Packages, services, kernel modules, network interfaces are all Nix expressions and all reproducible. [Colmena](https://github.com/zhaofengli/colmena) deploys across all nodes and produces VM images for Openstack/CAPO to consume.

### [Kubernetes](https://github.com/kubernetes/kubernetes)

Everything above the OS runs on Kubernetes. The OpenStack control plane, CAPI controllers, Sveltos, Crossplane are all Kubernetes workloads.

The baremetal OpenStack cluster runs kubeadm with [kube-vip](https://github.com/kube-vip/kube-vip) for API HA. The management cluster runs as CAPO-provisioned VMs, bootstrapped with kind then pivoted to self-manage. Workload clusters are provisioned declaratively by Cluster API.

[Kamaji](https://github.com/clastix/kamaji) offers an alternative for workload control planes: instead of 3 dedicated VMs per cluster, the API server, controller manager, and scheduler run as pods in the management cluster. A few pods instead of three VMs. Workers are still OpenStack VMs via CAPO.

{{< notice note "Fate-Sharing" >}}
Kamaji control planes share the management cluster's fate. A mgmt outage affects all Kamaji-hosted tenant control planes. For full VM isolation, `openstack-default` is available.
{{< /notice >}}

### [OpenStack](https://github.com/openstack)

Compute (Nova), networking (Neutron/OVN), storage (Cinder), images (Glance), identity (Keystone), load balancing (Octavia), DNS (Designate) are all running as Kubernetes operators via [Yaook](https://gitlab.com/yaook) and reconciled by Flux. No click-ops.

Adding compute is easy: join a node, apply labels, and Yaook operators wire it into Nova and OVN automatically. [Rook/Ceph](https://github.com/rook/rook) provides distributed storage. [Cilium](https://github.com/cilium/cilium) provides the CNI, replacing kube-proxy entirely.

![OpenStack Dashboard](/images/openstack-dashboard.png)

{{< notice tip "Scaling" >}}
No control plane re-provisioning needed, the capacity scales by labelling nodes.
{{< /notice >}}

### [Cluster API](https://github.com/kubernetes-sigs/cluster-api)

CAPO (Cluster API Provider for OpenStack) declaratively provisions workload clusters. Control planes, worker pools, networking are all Kubernetes CRs.

The ClusterClass pattern means a new cluster is a small `Cluster` CR with injected variables, this means no template forking needed. Sveltos-driven add-ons are gated by labels on the Cluster resource.

### [Flux](https://github.com/fluxcd/flux2)

Flux watches Argus and reconciles live state to match Git. Each cluster has its own sync path. On the OpenStack cluster, Flux manages everything: Cilium, Rook/Ceph, cert-manager, kgateway, [Crossplane](https://github.com/crossplane/crossplane), and the Yaook operators that run OpenStack itself.

Workload clusters get Flux installed via Sveltos (the Flux Operator + a FluxInstance pointing at Argus). Once installed, Flux self-reconciles itself and reconcile any other deployment set in Argus.

The core principle: **a change is a Git commit**.

### [Sveltos](https://github.com/projectsveltos/sveltos)

Sveltos runs on the management cluster and pushes add-ons to CAPI-provisioned workload clusters. Rather than forcing every add-on onto every cluster, opt-in `ClusterProfile` resources with label selectors control what each cluster receives.

Each add-on has its own ClusterProfile: Cilium bootstrap (with per-cluster values injected via Go templates from the CAPI Cluster resource), Flux Operator + FluxInstance, and OIDC RBAC. This means a shared Git path can never force an add-on onto a cluster that didn't opt in.

### Chihiro

**Chihiro** is a lightweight web UI for creating and managing workload clusters. Pick a Kubernetes version, set your worker groups and flavors, and Chihiro renders the `Cluster` CR and applies it to the management cluster.

![Chihiro UI — Create New Cluster](/images/chihiro-ui.png)

Chihiro is configured entirely through a ConfigMap: form fields are defined as injections (paths into the `Cluster` spec) and parameters (Go template variables). Adding a new field is a YAML edit so no code changes, no rebuild. It authenticates via OIDC against the shared Zitadel instance and lives on the management cluster.

## Design Principles

### Everything is Infrastructure as Code

There is no manual step. Every component from the operating system to the OpenStack control plane to tenant networking is described in code and reconciled automatically. A Git commit is the only interface.

{{< notice info "AI Assistance" >}}
Throughout this project, AI assistance has been used to increase velocity. As a hobby project, it helps move faster on the parts that matter. The fact that so much of RPCU is described as code also makes it far easier for AI to assist, everything is declarative, version-controlled, and self-documenting, so the context an AI needs lives right there in the repositories.
{{< /notice >}}

### Progressive Complexity

You don't need to understand the entire stack to get started. The NixOS layer is self-contained. Every node runs the same reproducible OS. The baremetal Kubernetes layer can run independently. The OpenStack control plane builds on top of Kubernetes. The management and workload clusters sit on top of OpenStack as VMs. Each layer has a clear boundary.

## The Hardware

RPCU runs on three [Hetzner dedicated servers](https://www.hetzner.com/dedicated-rootserver/) (Server Auction). Each machine has two NICs: one for public/management connectivity and a second internal NIC that connects all three nodes together on a dedicated Gbit LAN.

| Node      | CPU                         | RAM                    | Storage          | NIC                              |
| --------- | --------------------------- | ---------------------- | ---------------- | -------------------------------- |
| lucy      | Intel Core i7-8700 (6c/12t) | 64 GB DDR4 (4× 16 GB)  | 2× 1 TB NVMe SSD | 2× 1 Gbit (1 public, 1 internal) |
| makise    | Intel Core i7-7700 (4c/8t)  | 64 GB DDR4 (4× 16 GB)  | 2× 1 TB NVMe SSD | 2× 1 Gbit (1 public, 1 internal) |
| quinn     | Intel Core i7-8700 (6c/12t) | 128 GB DDR4 (4× 32 GB) | 2× 1 TB NVMe SSD | 2× 1 Gbit (1 public, 1 internal) |
| **Total** | **16 cores / 32 threads**   | **256 GB DDR4**        | **6 TB NVMe**    | **6× 1 Gbit**                    |

Modest consumer-grade hardware with no enterprise NICs, no redundant networking, no dedicated load balancers. That constraint is the point: everything above runs the full OpenStack control plane, Rook/Ceph storage, and nested CAPI clusters on top of it.

## What's Next

This post is an overview. In follow-up posts I'll detail the bootstrapping process more thoroughly, and over time I'll dig into some of the other technical details behind RPCU, the networking quirks, the OpenStack-on-Kubernetes wiring, and the GitOps workflow.

If you're interested in the architecture or want to try building something similar, the documentation and code are on [GitHub](https://github.com/rpcu). For the complete documentation of architecture deep-dives, bootstrap guides, and operational procedures, please visit the [RPCU Documentation](https://rpcu.github.io/aletheia/).

## The Team

RPCU is built by:

- [@aamoyel](https://github.com/aamoyel)
- [@Aifaidi](https://github.com/Aifaidi)
- [@Banh-Canh](https://github.com/Banh-Canh)
- [@clcondorcet](https://github.com/clcondorcet)
- [@maxduret](https://github.com/maxduret)
- [@chocomooncake](https://github.com/chocomooncake)

---

{{< notice note "A small personal note" >}}
[StackHPC](https://www.stackhpc.com/) inspired me to dig into OpenStack ahead of a role there. I figured the best way to prepare was to combine it with the tools I already know and enjoy and learn by building.
{{< /notice >}}
