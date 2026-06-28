---
title: 'Chihiro: A ConfigMap-Driven Web UI for Cluster API'
date: 2026-06-28
tags: ['infrastructure', 'kubernetes', 'cluster-api', 'gitops', 'go']
description: 'How Chihiro uses a ConfigMap to define its entire form, template the Cluster CR, and manage workload clusters without code changes.'
---

The [previous post](/posts/rpcu-bootstrap/) ended with a teaser: a follow-up on Chihiro, the web UI for creating and managing CAPI workload clusters. This is that post. It covers how Chihiro works under the hood, starting with the fact that the form itself is configuration, not code.

## The starting point

Before Chihiro, creating a workload cluster meant writing a `Cluster` manifest by hand and applying it with `kubectl`. That works fine if you are the only user and you have the YAML memorized. It does not scale to a team, and it does not inspire confidence in people who would rather click a button than parse a CRD schema.

Chihiro solves that. You pick a Kubernetes version, choose your worker group flavors and replica counts, toggle a few add-ons, and Chihiro renders the `Cluster` CR and applies it to the management cluster. OIDC auth. Kubeconfig generation. The standard quality of life things you would expect from a web UI.

<video controls width="100%">
  <source src="/videos/rpcu-bootstrap-demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

None of that is particularly novel. What sets Chihiro apart is how the form is built.

## The config is the form

Every field on the "Create Cluster" page is defined in a single ConfigMap. Not in Go code, not in a template file, not in a React component. In YAML that lives in the GitOps repo and gets reconciled by Flux.

### Injections: what Chihiro already knows

Chihiro has a set of built-in fields that map directly to paths in the `Cluster` spec. Cluster name goes to `metadata.name`. Kubernetes version goes to `spec.topology.version`. Control plane replicas goes to `spec.topology.controlPlane.replicas`. Worker groups go to `spec.topology.workers.machineDeployments`. These are the fields Chihiro always knows about, no configuration needed.

```yaml
injections:
  name:
    path: metadata.name
  version:
    path: spec.topology.version
    label: "Kubernetes Version"
    editable: true
  controlPlaneReplicas:
    path: spec.topology.controlPlane.replicas
    label: "Control Plane Replicas"
    editable: true
    min: 1
    max: 9
```

Worker groups are also an injection, but they are more complex because each cluster can have multiple machine deployment pools. Instead of a single value, they use a `worker_group_fields` schema for the form inputs and a `worker_group_template` to render each pool into a `machineDeployment` entry.

```yaml
worker_group_fields:
  flavor:
    label: "Flavor"
    type: select
    options: [small, medium, xmedium, large, xlarge]
    default: xmedium
    order: 2
  replicas:
    label: "Replicas"
    type: number
    default: "1"
    min: 1
    max: 10
    order: 3
worker_group_template: |
  name: {{ chihiro.field.name }}
  class: {{ chihiro.field.class }}
  replicas: {{ chihiro.field.replicas }}
  variables:
    overrides:
      - name: workerFlavor
        value: {{ chihiro.field.flavor }}
```

### Parameters: extending the form

Everything that is not an injection is a parameter. Custom fields that map to `{{ chihiro.<key> }}` placeholders in the cluster template. They support string, number, boolean, and select types, with optional defaults, constraints, and post-creation editing.

```yaml
parameters:
  podCIDR:
    label: "Pod CIDR"
    type: string
    default: "10.207.0.0/16"
  cilium:
    label: "Cilium CNI"
    type: boolean
    default: true
    true_value: enabled
    false_value: disabled
    editable: true
    path: "metadata.labels.'sveltos.argus.rpcu.io/cilium'"
```

The UI reads these definitions at runtime and renders whatever you give it. Want to expose a new cluster parameter? Edit the ConfigMap. Chihiro re-reads it on the next sync and the form updates. No code change, no rebuild, no redeploy of the application binary.

{{< notice info "Why this matters" >}}
Most web UIs hardcode their forms. Adding a field means writing frontend code, shipping a new image, and rolling it out. With Chihiro, adding a field is a YAML edit in a GitOps repo. Flux reconciles it, Chihiro picks it up, and the form changes for everyone. The same pattern that governs the rest of RPCU applies to the web UI itself.
{{< /notice >}}

## The template is Go

The cluster template is a Go template. Any `{{ chihiro.<key> }}` token is filled from the matching parameter or a built-in injection. The template defines the shape of the `Cluster` CR that Chihiro produces.

```yaml
template: |
  apiVersion: cluster.x-k8s.io/v1beta2
  kind: Cluster
  metadata:
    namespace: mgmt
    labels:
      sveltos.argus.rpcu.io/cilium: {{ chihiro.cilium }}
      type: workload
  spec:
    clusterNetwork:
      pods:
        cidrBlocks:
          - {{ chihiro.podCIDR }}
    topology:
      classRef:
        name: openstack-kamaji
      version: "0.0.0"
```

The `"0.0.0"` for version is intentional. Chihiro overwrites it from the injection at render time. The template does not need to know the version; it just needs to declare the structure.

Built-in tokens like `name`, `version`, `groups`, and `controlPlaneReplicas` are always available without declaring a parameter. Custom parameters fill everything else. When Chihiro renders the template, it resolves all tokens iteratively, validates that nothing is left unresolved, and produces a valid `Cluster` manifest.

## Constrained selects

A `select` parameter can have its options constrained by the value of another field. The classic use case in RPCU is the node image: different Kubernetes versions use different images, and you do not want someone picking an image that does not match their version.

```yaml
imageName:
  label: "Node Image"
  type: select
  options:
    - value: "hephaestus-kaas-25.11-{{ chihiro.version }}"
      label: "25.11"
      constrain:
        version: ["v1.35.4"]
    - value: "hephaestus-kaas-26.05-{{ chihiro.version }}"
      label: "26.05"
      constrain:
        version: ["v1.36.1"]
  editable: true
  path: "spec.topology.variables[2].value"
```

When you change the Kubernetes version, the image dropdown recomputes automatically. Only compatible options are shown. If you select an image that constrains the version to a single value, that value is pushed back to the version field. The dependency works in both directions.

This is not hardcoded logic. It is generic. You can constrain any select to any other field, built-in or custom. A driver field constrained to a region, a storage class constrained to a flavor, anything where the valid options depend on another choice. The constraint metadata in the ConfigMap is all Chihiro needs.

{{< notice note "Bidirectional coherence" >}}
When a select option pins a constrained field to a single value, Chihiro writes that value back. Changing version recomputes the image. Changing the image (which implies a specific version) recomputes the version. The system cascades: if the implied version triggers another dependent parameter, that recomputes too, all in the same update.
{{< /notice >}}

## Post-creation editing

Chihiro does not stop at cluster creation. Parameters marked `editable: true` with a `path` can be changed after the cluster exists. The path tells Chihiro where to write the value in the live `Cluster` resource.

```yaml
cilium:
  label: "Cilium CNI"
  type: boolean
  default: true
  true_value: enabled
  false_value: disabled
  editable: true
  path: "metadata.labels.'sveltos.argus.rpcu.io/cilium'"
```

Toggle Cilium off in the UI, and Chihiro patches the label on the Cluster resource. Sveltos sees the label change and removes the Cilium add-on. The same flow works for OIDC RBAC, OpenStack integration labels, or any custom parameter you expose with an `editable` and `path`.

The kubeconfig download, version upgrades, worker group resizing, and control plane scaling all work the same way: check permissions, validate limits, write the value at the configured path.

## Add-ons via labels

The boolean parameters in Chihiro are not just form toggles. They write labels to the Cluster resource, and anything watching those labels can react. Sveltos is what RPCU uses, but the mechanism is generic. Flux, custom controllers, operators, any reconciler that watches Cluster labels or annotations can pick up the changes Chihiro makes.

The template in the ConfigMap writes labels like this:

```yaml
labels:
  sveltos.argus.rpcu.io/cilium: {{ chihiro.cilium }}
  sveltos.argus.rpcu.io/oidc-rbac: {{ chihiro.oidcRbac }}
  sveltos.argus.rpcu.io/openstack-integration: {{ chihiro.openstack }}
  type: workload
```

In RPCU, Sveltos ClusterProfiles match those labels with a `clusterSelector`:

```yaml
apiVersion: config.projectsveltos.io/v1beta1
kind: ClusterProfile
metadata:
  name: cilium
spec:
  syncMode: ContinuousWithDriftDetection
  clusterSelector:
    matchLabels:
      type: workload
      sveltos.argus.rpcu.io/cilium: enabled
```

Check the "Cilium CNI" box in Chihiro, the label becomes `enabled`, Sveltos deploys Cilium. Uncheck it, the label becomes `disabled`, Sveltos removes it. Chihiro does not know what Cilium is or how it gets installed. It just writes a label. The reconciler does the rest.

{{< notice info "Opt-in by design" >}}
Every add-on is gated by its own label. A shared Git path never forces an add-on onto a cluster that did not opt in. One label per add-on. Changing what an add-on does, adding a new one, or adjusting Helm values is a separate commit that never touches Chihiro's ConfigMap.
{{< /notice >}}

## Controlling who sees what

Every parameter supports `visible_groups`, a list of OIDC groups that can see and edit it. Leave it empty and everyone with create access sees it. Set it to `[kube-admin]` and only admins see it.

```yaml
oidcRbac:
  label: "OIDC RBAC"
  type: boolean
  editable: true
  visible_groups:
    - kube-admin
```

This means the same ConfigMap serves all users. Regular users see a simplified form. Admins see the full set of toggles. The frontend fetches the filtered parameter list per user and renders accordingly.

Three authorization tiers govern the whole system. Admins can do everything. Creators can create and edit clusters they own or share a group with. Everyone else gets read-only access to clusters matching their groups.

## How to extend it

The pattern for adding a new field is the same every time:

1. Declare a parameter in the ConfigMap with a label, type, and default.
2. Reference it in the template as `{{ chihiro.newField }}`.
3. If it should be editable after creation, add `editable: true` and a `path`.

That is it. The form picks it up on the next sync. No Go code, no frontend changes, no image rebuild. For RPCU this means Chihiro evolves alongside the infrastructure. New Cluster API variables, new add-on toggles, new flavor options, all land as ConfigMap edits in the same GitOps workflow that manages everything else.

If you run a different CAPI provider, you swap the template. If you need different flavors, you change the select options. If you want to expose a field only to certain groups, you add `visible_groups`. The ConfigMap schema is the interface, and it is a deliberately narrow one.

{{< notice tip "Try it yourself" >}}
The full ConfigMap is in [Argus](https://github.com/RPCU/argus) under `clusters/mgmt/apps/chihiro/`. Clone it, change a parameter, and watch the form update. The source code for Chihiro itself lives at [github.com/Bealvio/chihiro](https://github.com/Bealvio/chihiro). Full documentation is in [Aletheia](https://rpcu.github.io/aletheia/workload-clusters/chihiro.html).
{{< /notice >}}

