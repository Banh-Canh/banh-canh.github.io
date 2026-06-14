# AGENTS.md

Guidance for AI agents working in this repository.

## What this is

A personal blog by Victor Hang (Senior Technical Lead, infrastructure/platforms),
built with **Hugo** and the **hugo-coder** theme, deployed to **GitHub Pages**.

- **Static site generator**: Hugo (extended), pinned to `0.162.1` in CI.
  Note: the `hugo-coder` theme uses the newer `layouts/_shortcodes/`,
  `_partials/`, `_markup/` layout convention, which requires Hugo **≥ 0.146**.
  An older local Hugo (e.g. 0.126) will fail with
  `template for shortcode "notice" not found` — use a current Hugo to build.
- **Theme**: [`hugo-coder`](https://github.com/luizdepra/hugo-coder), a git submodule at `themes/hugo-coder`
- **Hosting**: GitHub Pages at `https://banh-canh.github.io/`
- **Repo**: `git@github.com:Banh-Canh/banh-canh.github.io.git` (branch `main`)

## Repository layout

```
hugo.toml              # Site configuration (baseURL, theme, params, menu, social)
archetypes/
  default.md           # Template for `hugo new` (TOML +++ frontmatter, draft = true)
content/
  about.md             # About page (YAML frontmatter)
  posts/
    _index.md          # Posts list page intro
    *.md               # Blog posts
themes/hugo-coder/     # Theme (git submodule — do NOT edit directly)
.github/workflows/
  hugo.yml             # Build + deploy to GitHub Pages on push to main
assets/ data/ i18n/ static/  # Currently empty; standard Hugo dirs
public/ resources/     # Build output / Hugo cache — do NOT edit or commit by hand
```

## Conventions

### Writing posts

- Posts live in `content/posts/<slug>.md`.
- **Existing posts use YAML frontmatter** (`---` delimiters), e.g.:
  ```yaml
  ---
  title: "Post Title"
  date: 2026-06-14
  tags: ["infrastructure", "kubernetes"]
  description: "One-line summary used in listings and meta tags."
  ---
  ```
  Match this style when adding or editing posts. Note: the `archetypes/default.md`
  template emits TOML (`+++`) with `draft = true` — prefer the YAML style of the
  existing posts for consistency, and remove/keep `draft` intentionally (a draft
  is not published).
- Use ATX headings starting at `##` (the `title` is the H1). Keep prose wrapped
  naturally; the theme renders standard CommonMark.
- `tags` feed the `tags` taxonomy; `category`/`categories` is also configured.
- Tone: technical, direct, no fluff. Prioritize accuracy over marketing language.

### Voice & accuracy

This is a technical blog. When a post describes real systems (e.g. the RPCU
infrastructure posts), the content **must match the actual implementation**.
Cross-check claims against the source repositories before writing or editing.

## The RPCU posts — source of truth

`content/posts/rpcu-architecture.md` documents the **RPCU** infrastructure
project, which lives in sibling repositories (relative to this repo):

- `../argus` — GitOps repo (Flux CD). Has a detailed `AGENTS.md` describing every
  component, cluster, and dependency. **This is the authoritative source** for
  the GitOps/Kubernetes/OpenStack/CAPI architecture.
- `../hephaestus` — NixOS operating system (Colmena/Ginx) for the **baremetal**
  nodes. Has an `AGENTS.md` describing node agents (kubelet, kube-vip, kubeadm,
  netbird, cloud-init, etc.) and MTU/networking notes.
- `../aletheia` — VitePress documentation site (`introduction.md`, `index.md`).

### Critical architecture facts (do not get these wrong)

- The **OpenStack cluster runs on baremetal** NixOS nodes: `lucy`, `makise`,
  `quinn` (managed by Hephaestus; HA Kubernetes API via kube-vip). It hosts the
  OpenStack control plane via **Yaook operators**, Rook/Ceph storage, Cilium
  (with L2-announcement LoadBalancer), and **owns the shared Zitadel
  configuration** (via Crossplane). Zitadel itself is a **SaaS instance**, not
  self-hosted — the cluster manages the org/projects/roles/OIDC clients against
  it, it does not run the Zitadel server.
- The **management cluster runs as VMs ON OpenStack** (CAPO-provisioned).
  Bootstrapped with kind + `clusterctl`, intended to self-manage after
  `clusterctl move`. It uses the **OpenStack CCM + Octavia** for LoadBalancer
  Services and **Cinder CSI** for storage — NOT Cilium L2 / Rook. It runs
  Cluster API, Sveltos, and a small Crossplane install (only the `chihiro` OIDC
  client; it must NOT manage the shared Zitadel platform).
- **Workload clusters** are also CAPI-provisioned OpenStack VMs, configured
  per-cluster by opt-in **Sveltos `ClusterProfile`s** (labels gate Flux, Cilium,
  OIDC RBAC). There is no `clusters/workload/` directory.
- Two scaling paths: **label a baremetal node** to grow OpenStack (Yaook
  schedules `nova-compute`/`ovn-controller`); **scale a `MachineDeployment`** to
  grow a CAPI cluster.

A common mistake to avoid: depicting the management cluster as the *base* that
"provisions and manages the OpenStack cluster." It is the reverse — OpenStack is
the foundation, and the management cluster sits on top of it as VMs.

When editing RPCU posts, re-read `../argus/AGENTS.md` and `../hephaestus/AGENTS.md`
to verify component names, versions, and relationships.

## Local development

```bash
hugo server -D          # Live preview including drafts (http://localhost:1313)
hugo --gc --minify      # Production build into ./public
hugo new posts/<slug>.md  # Scaffold a new post from the archetype
```

The theme is a submodule — after cloning, run:

```bash
git submodule update --init --recursive
```

## Build & deploy

- CI: `.github/workflows/hugo.yml` runs on push to `main` (and manual dispatch).
- It installs Hugo extended `0.162.1`, checks out submodules recursively, builds
  with `--gc --minify`, and deploys `./public` to GitHub Pages.
- The `baseURL` is overridden at build time to the Pages URL; the value in
  `hugo.toml` is the canonical site URL.

## Do / Don't

- **Do** match the YAML frontmatter style of existing posts.
- **Do** verify technical claims about RPCU against `../argus`, `../hephaestus`,
  `../aletheia` before publishing.
- **Don't** edit files under `themes/hugo-coder/` (submodule) or `public/` /
  `resources/` (generated).
- **Don't** commit unless explicitly asked. Show a diff and the list of files
  first.
- **Don't** bump the Hugo version in CI without a reason.
