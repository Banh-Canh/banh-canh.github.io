---
title: 'Bootstrapping RPCU: From Bare Metal to a Full Cloud Platform'
date: 2026-06-22
tags: ['infrastructure', 'openstack', 'kubernetes', 'nixos', 'gitops']
description: 'How RPCU goes from three bare servers to a fully operational OpenStack cloud — what you run, in what order, and where the manual steps are.'
---

In the [previous post](/posts/rpcu-architecture/) I covered the architecture: three clusters, two parts, a stack of open source tools wired together with Infrastructure as Code. Full documentation lives in [Aletheia](https://github.com/RPCU/aletheia). This post is about the actual bootstrap. What you run, in what order, and where the manual steps are.

Fewer steps than you'd expect. The complexity lives in the definitions, not in the process.

## The Two Parts

```
Part 1: Baremetal OpenStack cluster (manual)
  ISO → NixOS install → kubeadm → Cilium → Flux → Yaook operators → OpenStack

Part 2: CAPI management cluster (Flux-driven)
  qcow2 (kaas profile) → kind + Flux → CAPO provisions VMs → clusterctl move → Flux self-managing
```

The OpenStack cluster is the foundation. Three Hetzner servers, no CAPI, no external orchestrator. The management cluster runs as VMs on top of it.

## Part 1: The Baremetal OpenStack Cluster

### Step 1: Build and install NixOS

[Hephaestus](https://github.com/RPCU/hephaestus) provides a devenv with build helpers:

```bash
devenv shell
build-iso --argstr partition root
```

Two image builders matter here: `build-iso` for baremetal, `build-qcow2` for VMs. Same NixOS configuration, different output formats. The partition layout is configurable: `root`, `root-grub` for BIOS, `small` for constrained disks. There is also `test-iso` which builds and boots the ISO in QEMU locally.

Write the ISO to a USB drive, plug it into a server, boot. The installer auto-detects the first unused disk, wipes it, partitions it, runs `nixos-install`, and reboots. After first boot, the [`ginx`](https://github.com/didactiklabs/ginx) agent pulls the latest config from Git and applies it via `colmena apply-local`. Repeat for each node (lucy, makise, quinn).

{{< notice info "Hephaestus's perimeter" >}}
Hephaestus owns the operating system. Packages, services, kernel modules, network interfaces, all Nix expressions. It does not know about Kubernetes or OpenStack. Its job ends at a running NixOS node with SSH access.
{{< /notice >}}

### Step 2: Bootstrap Kubernetes

SSH into the first node and run a single command:

```bash
sudo initKubeadm
```

`initKubeadm` and `joinCPKubeadm` are helper scripts produced by the NixOS configuration in Hephaestus. The kubeadm config and all related variables (API server address, networking, kubelet flags, versions) are Nix templates evaluated at build time. When you run `build-iso` or `colmena apply`, the entire OS image including the Kubernetes configuration is assembled and version-locked. No runtime config files to hand-edit, no templates to render.

Copy the join output from `initKubeadm`. On the other two nodes:

```bash
sudo joinCPKubeadm <TOKEN> <CERT_KEY>
```

Three commands across three nodes, and you have an HA Kubernetes cluster with kube-vip providing a floating API endpoint.

### Updating nodes

Updates don't require reinstalling. NixOS is declarative, so a node update is just:

```bash
colmena apply --on lucy
```

Colmena builds the new system closure on the target, switches to it, and reboots if needed. This is also how `ginx` works in the background: it pulls the latest Hephaestus repo and runs `colmena apply-local` automatically.

### Step 3: Install Cilium

Cilium replaces kube-proxy and provides eBPF networking plus L2 LoadBalancer. One flag stands out: `socketLB.hostNamespaceOnly: true`. Required because the nodes host nested KVM/QEMU OpenStack VMs, limiting socket-LB to the host namespace prevents the eBPF load-balancer from interfering with traffic inside guest VMs.

{{< notice info "Kubernetes perimeter" >}}
Kubernetes is the platform layer. kubeadm bootstraps the cluster, kube-vip provides HA, Cilium handles networking. This layer knows nothing about OpenStack. It is a standard Kubernetes cluster that happens to run on baremetal. Everything above this step is a workload.
{{< /notice >}}

### Step 4: Install Flux and let Git take over

Install the Flux Operator, then apply a FluxInstance pointing at `clusters/openstack/` in the [Argus](https://github.com/RPCU/argus) repository. Flux reconciles the full stack: cert-manager, gateway components, Rook/Ceph, Crossplane, External Secrets, and the Yaook operators that run OpenStack itself.

The reconciliation order matters. Cilium is already running. Then cert-manager and gateways. Then Rook/Ceph for storage. Then Yaook operators deploying Keystone, Nova, Neutron/OVN, Glance, Cinder, Octavia, Designate, Barbican. Once all Kustomizations reach `Ready`, the OpenStack control plane is live.

{{< notice info "Yaook's perimeter" >}}
Yaook operators own the OpenStack control plane. Each operator manages one service as a Kubernetes CRD. They do not manage Kubernetes itself, the CNI, or storage. Their job is to run OpenStack on top of an existing cluster.
{{< /notice >}}

## Part 2: The CAPI Management Cluster

The management cluster runs as OpenStack VMs provisioned by CAPO. It cannot bootstrap itself from nothing — it needs a temporary kind cluster. Flux drives the CAPI stack on both clusters. Same source, same reconciliation.

The VMs use a qcow2 disk image uploaded to Glance, built with:

```bash
build-qcow2 --argstr profile kaas
```

The `kaas` profile (Kubernetes-as-a-Service) is the cloud variant: cloud-init, QEMU guest agent, DHCP networking, minimal partition layout for OpenStack volumes. Same NixOS base, tuned for VMs. There is also `build-oci-qcow2` which wraps the qcow2 in an OCI container image for easy Glance upload.

### Step 1: Create a kind cluster and install Flux

Spin up a bare kind cluster and install the Flux Operator. That's the entire bootstrap plane.

### Step 2: Point Flux at the management cluster path

Apply a FluxInstance watching `clusters/mgmt/` in Argus. Flux reconciles the CAPI stack: cert-manager, the Cluster API Operator, CAPI/kubeadm/CAPO providers, ORC, ClusterClass templates, and credential plumbing. No `clusterctl init`, no manual `helm install`. Everything comes from Git.

### Step 3: Create the capo-variables secret

The one thing that cannot be in Git. OpenStack admin `clouds.yaml`:

```bash
kubectl --context kind-capi-mgmt -n capo-system create secret generic capo-variables \
  --from-file=clouds.yaml=<path-to-clouds.yaml>
```

Three Flux Kustomizations depend on this secret, and they all use `wait: false` so a missing secret doesn't block the rest.

### Step 4: Create the target cluster

Once Flux has reconciled the CAPI providers and ClusterClass, apply the Cluster CR. CAPO provisions OpenStack VMs, kubeadm bootstraps Kubernetes on them.

### Step 5: Pivot with clusterctl move

`clusterctl move` transfers all CAPI resources from the temporary kind cluster to the target management cluster. After the move, the management cluster owns its own CAPI lifecycle.

### Step 6: Wire up GitOps on the management cluster

Delete the kind cluster, install Cilium and Flux on the management cluster, create the `capo-variables` secret again. The FluxInstance points at the same `clusters/mgmt/` path, so Flux picks up everything the kind cluster was reconciling.

{{< notice info "CAPI perimeter" >}}
Cluster API owns workload cluster lifecycle. It provisions VMs via CAPO, bootstraps Kubernetes via kubeadm, and manages cluster upgrades and scaling. CAPI does not manage the OpenStack control plane, the CNI, or storage. Its job ends at a running Kubernetes cluster.
{{< /notice >}}

From here, Flux reconciles the remaining components: CCM via Octavia, Cinder CSI, ExternalDNS, Crossplane, Sveltos, and the chihiro web UI.

## Where the Manual Steps Are

The bootstrap is mostly automated, but a few things require human input.

### The `capo-variables` secret

Created manually on the kind cluster, then again on the management cluster after pivot. Contains OpenStack admin credentials. Not in Git, not managed by Flux. This is the root secret from which all other OpenStack credentials derive via External Secrets Operator, feeding CAPO, the CCM, Cinder CSI, and ExternalDNS.

### OpenStack resource IDs

OpenStack resources generate UUIDs at creation time: external networks, Ceph secrets, security groups, projects. These IDs are hardcoded in the GitOps configuration because they cannot be known before the cluster exists. You create the resources once, look up their IDs, and enter them into the appropriate YAML files. A one-time manual step that feeds an otherwise fully declarative pipeline.

## Why It Works

The bootstrap is short because each technology has a clearly defined perimeter:

- **NixOS** (Hephaestus) owns the operating system. It does not know about Kubernetes or OpenStack.
- **Kubernetes** (kubeadm, Cilium, kube-vip) owns the platform layer. It does not know about OpenStack.
- **OpenStack** (Yaook) owns the cloud control plane. It runs on top of Kubernetes, not inside it.
- **Cluster API** (CAPI/CAPO) owns workload cluster lifecycle. It does not manage the OpenStack control plane.

The bootstrap is wiring them together in the right order. You can inspect any layer independently, debug it independently, and replace it independently.

## Up next: Chihiro

So far, creating CAPI workload clusters means writing manifests and talking to `kubectl`. **Chihiro** fixes that — a lightweight dashboard that watches CAPI resources and lets you spin up, edit, and tear down clusters from a web interface. A video walkthrough is coming with the release.
