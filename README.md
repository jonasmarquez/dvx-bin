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

## Requirements

- macOS or Linux (x86_64 or arm64).
- A POSIX-compatible shell (bash, zsh, etc.).
- For Kubernetes-related commands (`dvx k8s …`):
  - `kubectl` installed and available in your `PATH`.
  - A working kubeconfig for the clusters you want to use with dvx.
- For Homebrew installation: Homebrew installed on your system.
- An editor configured in `$DVX_EDITOR` or `$EDITOR` (vim, nvim, code, etc.) for `dvx dim edit` (optional but recommended).

> 🔎 If `kubectl` is not present, dvx will still work for dimensions, env and secrets,
> but `dvx k8s *` commands will fail with a clear error message.

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

    dvx k8s inspect          # dashboard-like view of resources for the current namespace
    dvx k8s count            # numeric summary of resources in the current namespace
    ```

  - `dvx k8s env` & `dvx k8s ns` work on the dvx runtime kubeconfig, so you can
    safely experiment with namespaces/contexts without touching your original
    kubeconfig.
  - `dvx k8s inspect` and `dvx k8s count` respect the effective namespace in this order:
    1. Explicit argument (`dvx k8s inspect <ns>`, `dvx k8s count <ns>`)
    2. Namespace set on the dvx runtime kubeconfig (e.g. via `dvx k8s ns`)
    3. `KUBE_NAMESPACE` from the shell environment
    4. `kubernetes.namespace` from `dimensions.yaml`
    5. `default`
  - The Kubernetes UI layer:
    - Shared **ASCII section boxes** for `inspect`/`count` with context & namespace:
      ```text
      ┌─ ☸ Pods ─────────────────────────────────────────────
      │   ctx: dc1-dev-platform   ns: platform
      └──────────────────────────────────────────────────────
      ```
    - Internal color handling for table headers and statuses:
      - Pod phases (Running, Pending, Failed, Succeeded/Completed, Unknown).
      - Job statuses (Failed, Complete/Completed/Succeeded) and their completions (e.g. `0/1`, `1/1`).
    - Colors are applied directly by dvx, so no external tools like `kubecolor` are required.

- 🎛️ **TTY-aware UX**
  - When running in a real terminal, dvx uses an interactive picker
    (via `promptui`) for:
    - Dimension selection (`dvx dim pick`)
    - Kubernetes dimension selection (`dvx k8s pick`)
    - Namespace selection (`dvx k8s ns` without args)
    - Secrets listing and actions (`dvx secrets list`, `dvx secrets gen`)
  - When used in scripts / CI (no TTY), dvx falls back to classic, non-interactive output.
  - You can disable colors by setting either:
    - `NO_COLOR` (standard convention), or
    - `DVX_NO_COLOR=1` (dvx-specific override).

---

## Installation

### 1. Homebrew (recommended on macOS & Linux)

Tap and install:

```bash
brew tap jonasmarquez/dvx
brew install dvx
```

Upgrade:

```bash
brew update
brew upgrade dvx
```

Check the installed version:

```bash
dvx version
```

You should see something like:

```text
dvx ⚙️  version 0.15.0
```

### 2. Manual download

Go to the **Releases** page and download the appropriate tarball:

- `dvx_0.15.0_darwin_arm64.tar.gz`   – macOS Apple Silicon (M1/M2/M3)
- `dvx_0.15.0_darwin_amd64.tar.gz`   – macOS Intel
- `dvx_0.15.0_linux_amd64.tar.gz`    – Linux x86_64
- `dvx_0.15.0_linux_arm64.tar.gz`    – Linux ARM64

Example (macOS arm64):

```bash
curl -L -o dvx_0.15.0_darwin_arm64.tar.gz   https://github.com/jonasmarquez/dvx-bin/releases/download/v0.15.0/dvx_0.15.0_darwin_arm64.tar.gz

tar xzf dvx_0.15.0_darwin_arm64.tar.gz
chmod +x dvx
sudo mv dvx /usr/local/bin/  # or any directory in your $PATH
```

Verify:

```bash
dvx version
```

---

## Quick start

### 1. Initialize dvx

```bash
dvx init
```

This will:

- Create the config directory (usually `~/.config/dvx`)
- Create a starter `dimensions.yaml` if it doesn’t exist
- Optionally initialize the secrets vault

You can inspect the starter config:

```bash
cat ~/.config/dvx/dimensions.yaml
```

…and then edit it:

```bash
dvx dim edit
# or edit the file directly:
# $EDITOR ~/.config/dvx/dimensions.yaml
```

### 2. Define your first dimension

In `dimensions.yaml` you will see an example like:

```yaml
dimensions:
  - name: work-dc1-dev-platform
    type: work
    org: entity
    site: dc1
    environment: dev
    project: platform-alpha
    description: "Example dimension for a dev environment in DC1."

    env:
      DVX_ORG: "entity"
      DVX_SITE: "dc1"
      DVX_ENVIRONMENT: "dev"
      DVX_PROJECT: "platform-alpha"
      PROJECT_ROOT: "~/DEV/entity/platform-alpha"
      TERRAFORM_ROOT: "${PROJECT_ROOT}/infra"

    kubernetes:
      enabled: true
      kubeconfig: "~/.kube/config-dc1-dev"
      context: "dc1-dev-platform"
      namespace: "platform"

    terraform:
      root_dir: "${PROJECT_ROOT}/infra"
      workspace: "dc1-dev"
      extra_env:
        TF_VAR_site: "${DVX_SITE}"
        TF_VAR_env: "${DVX_ENVIRONMENT}"

    cloud:
      aws:
        profile: "entity-dev"
      azure:
        subscription_id: "secret:azure.subscriptions.work-dev"
      gcp:
        project_id: "secret:gcp.projects.work-dev"
```

You can create more dimensions following the same structure.

### 3. Pick and use a dimension

List dimensions:

```bash
dvx dim ls
```

Interactively pick one:

```bash
dvx dim pick
```

Show current:

```bash
dvx dim current
```

Export its environment into your shell:

```bash
eval "$(dvx env)"
```

Now all the variables (including `DVX_*`, `PROJECT_ROOT`, `TF_VAR_*`, etc.)
are loaded in your current shell session.

---

## Secrets vault

dvx includes a local encrypted vault to store secrets referenced in your
dimension configuration.

### Initialize the vault

```bash
dvx secrets init
```

You’ll be prompted for a **master password** (twice).  
This creates `~/.config/dvx/secrets.enc.yaml`, encrypted with AES-GCM and a key
derived from your password using `scrypt`.

### Store a secret

```bash
dvx secrets set azure.subscriptions.work-dev
```

You’ll enter the master password, then the value for that secret.

### Use the secret in your config

In `dimensions.yaml`:

```yaml
cloud:
  azure:
    subscription_id: "secret:azure.subscriptions.work-dev"
```

When you run:

```bash
eval "$(dvx env)"
```

dvx will:

1. Detect all `secret:<id>` references
2. Prompt once for the master password (per dvx process)
3. Replace them with the decrypted secret values in your environment

### Generate secrets

```bash
dvx secrets gen
```

You can interactively generate:

- Passwords
- API tokens (URL-safe)
- Hex tokens
- UUID v4
- Erlang cookies
- Passphrases (multi-word)

…and then choose whether to:

- Save them into the dvx vault
- Export as an `export MY_VAR=...` line
- Just print the value (plain + base64, useful for Kubernetes Secrets)

---

## Kubernetes helpers

dvx reads the `kubernetes` block from your dimension and exposes helpers:

```bash
dvx k8s current
```

Shows:

- Whether Kubernetes is enabled for this dimension
- Kubeconfig file (original path from `dimensions.yaml`)
- Context
- Namespace

Export only K8s-related variables (using the dvx **runtime** kubeconfig):

```bash
eval "$(dvx k8s env)"
# Now KUBECONFIG points to ~/.config/dvx/kube/<dimension>.kubeconfig
# KUBE_CONTEXT and KUBE_NAMESPACE are set when defined
```

Open a subshell with the full dimension env (including K8s):

```bash
dvx k8s shell
# type 'exit' or Ctrl+D to return
```

Pick the Kubernetes dimension interactively:

```bash
dvx k8s pick
```

Select or change the namespace for the current dimension:

```bash
# Interactive namespace picker for the current dimension
dvx k8s ns

# Non-interactive (set directly)
dvx k8s ns kube-system

# You can combine it with eval to update KUBE_NAMESPACE in your shell:
eval "$(dvx k8s ns kube-system)"
```

Inspect resources for the current namespace with a dashboard-like view:

```bash
dvx k8s inspect
dvx k8s inspect kube-system
```

Get a numeric summary of resources for the current namespace:

```bash
dvx k8s count
dvx k8s count kube-system
```

All these commands operate on the **runtime kubeconfig** managed by dvx,
keeping your original kubeconfig untouched.

---

## Security notes

- The secrets vault is stored locally in:
  - `~/.config/dvx/secrets.enc.yaml`
- Encryption:
  - AES-GCM (authenticated encryption)
  - Key derived with `scrypt` from your master password + random salt
- dvx never stores your master password.
- Secret values are resolved in memory at runtime and exported into your shell
  only when you explicitly call commands like `dvx env` or `dvx k8s env`.

You are still responsible for:

- Protecting your master password
- Protecting your machine and home directory
- Avoiding pasting sensitive values into logs or shared terminals

---

## Usage examples

```bash
# Show version
dvx version

# Initialize config + optional vault
dvx init

# Work with dimensions
dvx dim ls
dvx dim pick
dvx dim show
dvx dim use work-dc1-dev-platform
dvx dim current
dvx dim edit

# Secrets
dvx secrets init
dvx secrets set github.token.entity
dvx secrets get github.token.entity
dvx secrets list
dvx secrets gen

# Environment (full dimension)
dvx env
dvx env list
eval "$(dvx env)"

# Kubernetes (runtime-based)
dvx k8s current
dvx k8s env
eval "$(dvx k8s env)"

dvx k8s ns
dvx k8s ns kube-system
eval "$(dvx k8s ns kube-system)"

dvx k8s inspect
dvx k8s inspect kube-system

dvx k8s count
dvx k8s count kube-system

dvx k8s shell
dvx k8s pick
```

---

## License

`dvx` is distributed under the MIT license.  
See the main repository for full source code and details.
