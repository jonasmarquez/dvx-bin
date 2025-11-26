# dvx – Dev eXperience helper CLI

`dvx` is a small CLI tool that helps you manage your **work “dimensions”**:
logical contexts that group together:

- Organization / site / environment / project
- Environment variables
- Kubernetes cluster (kubeconfig + context + namespace)
- Terraform workspace + extra env
- Cloud accounts (AWS, Azure, GCP)
- Encrypted secrets (with a local vault)

Think of a **dimension** as your “current work universe”:  
`work @ dc1 / dev / on-prem / project X` — and you can switch between them quickly.

This repository **only hosts prebuilt binaries and the Homebrew tap**.  
The source code and detailed docs live in the main `dvx` repository.

---

## Features at a glance

- 📁 **Dimensions**
  - Define multiple work contexts in `~/.config/dvx/dimensions.yaml`
  - Switch quickly: `dvx dim pick` or `dvx dim use <name>`
  - See current context: `dvx dim current`
  - Edit with your favorite editor: `dvx dim edit`

- 🔐 **Encrypted secrets vault**
  - Local, encrypted vault (`~/.config/dvx/secrets.enc.yaml`)
  - AES-GCM + scrypt key derivation
  - `dvx secrets init / set / get / list / delete / purge / passwd`
  - Secret generator: `dvx secrets gen` (passwords, API tokens, UUIDv4, Erlang cookie, passphrases)
  - Use secret references in your dimension config: `secret:my.secret.id`

- 🌍 **Environment management**
  - Build environment variables from your dimension:
    - Global `env` block
    - `terraform.extra_env` block
    - Automatic `DVX_*` metadata (`DVX_DIM_NAME`, `DVX_ORG`, etc.)
  - Expand `~`, `$VAR`, `${VAR}` inside values
  - Export to your shell:
    ```bash
    eval "$(dvx env)"          # for current dimension
    eval "$(dvx env work-dev)" # for a specific dimension
    ```
  - Inspect env as `KEY=VALUE` without changing your shell:
    ```bash
    dvx env list
    ```

- ☸️ **Kubernetes helpers (with per-dimension runtime kubeconfig)**
  - Per-dimension Kubernetes config in `dimensions.yaml`:
    - `kubernetes.enabled`
    - `kubeconfig`, `context`, `namespace`
  - dvx maintains a **runtime kubeconfig** per dimension under:
    - `~/.config/dvx/kube/<dimension>.kubeconfig`
  - The original kubeconfig you reference in `dimensions.yaml` is **never modified**;
    all context/namespace changes happen on the dvx runtime copy.

  - Commands:
    ```bash
    dvx k8s current          # show K8s info for current dimension
    dvx k8s env              # export only K8s-related vars (runtime kubeconfig)
    eval "$(dvx k8s env)"    # apply KUBECONFIG/KUBE_CONTEXT/KUBE_NAMESPACE

    dvx k8s ns               # pick a namespace interactively (per dimension)
    dvx k8s ns <name>        # set a namespace non-interactively
    dvx k8s shell            # open a subshell with the dimension's env
    dvx k8s pick             # interactive picker of K8s-enabled dimensions
    ```

  - `dvx k8s env` & `dvx k8s ns` work on the dvx runtime kubeconfig, so you can
    safely experiment with namespaces/contexts without touching your original
    kubeconfig.

- 🎛️ **TTY-aware UX**
  - When running in a real terminal, dvx uses an interactive picker
    (via `promptui`) for:
    - Dimension selection (`dvx dim pick`)
    - Kubernetes dimension selection (`dvx k8s pick`)
    - Namespace selection (`dvx k8s ns` without args)
    - Secrets listing and actions (`dvx secrets list`, `dvx secrets gen`)
  - When used in scripts / CI (no TTY), dvx falls back to classic, non-interactive output.

---

## Installation

### 1. Homebrew (recommended on macOS & Linux)

Tap and install:

```bash
brew tap jonasmarquez/dvx
brew install dvx