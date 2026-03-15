# Core Terraform Workflow

### **SECTION 1: THE TERRAFORM WORKFLOW**

*Write, Init, Plan, Apply, Destroy.*

### **1. The Core Loop**

- **The Rule:** Every Terraform operation follows the same fundamental lifecycle: **Write → Init → Plan → Apply**. Optionally **→ Destroy**.
- This workflow applies whether you are working solo or in a team with CI/CD.

**Step-by-Step:**

| **Step** | **Command** | **What It Does** |
| --- | --- | --- |
| **Write** | (editor) | Author `.tf` configuration files defining desired infrastructure |
| **Init** | `terraform init` | Download providers, initialize backend, install modules |
| **Plan** | `terraform plan` | Preview what Terraform **will** change (dry run, no mutations) |
| **Apply** | `terraform apply` | Execute the changes to reach desired state |
| **Destroy** | `terraform destroy` | Tear down all managed resources |

**Key Concepts:**

- **Declarative Model:** You declare the **desired end state**. Terraform figures out the steps to get there.
- **Idempotent:** Running `apply` multiple times with no config changes results in **no changes**. Terraform only acts on drift.
- **Plan Before Apply:** `terraform apply` generates a plan and prompts for confirmation **unless** you pass `-auto-approve` or a saved plan file.

*Exam Trigger:* "What is the correct order of Terraform commands for a new project?" → `init` → `plan` → `apply`.

---

### **SECTION 2: CLI COMMANDS DEEP DIVE**

*Every command, flag, and nuance the exam tests.*

### **1. terraform init**

**The Rule:** **Must run first** before any other command. Initializes the working directory.

**What It Does:**

- Downloads **provider plugins** to `.terraform/providers/`.
- Initializes the **backend** (local or remote state storage).
- Downloads **modules** referenced in configuration.
- Creates the `.terraform.lock.hcl` **dependency lock file** (pins provider versions).

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-upgrade` | Upgrade providers/modules to latest versions within constraints |
| `-migrate-state` | Migrate state to a new backend configuration |
| `-reconfigure` | Reconfigure backend, ignoring any existing state migration |
| `-backend-config=KEY=VALUE` | Pass backend config values at runtime (partial configuration) |

**Key Facts:**

- **Safe to re-run.** Running `init` multiple times is harmless.
- `-migrate-state` and `-reconfigure` are **mutually exclusive**. You use one or the other when changing backends.
- `-backend-config` is used for **partial backend configuration** (e.g., passing secrets not stored in code).

*Exam Trigger:* "Provider not found" or "Backend not initialized" → Run `terraform init`.

*Exam Trap:* `terraform init` does **not** create infrastructure. It only prepares the working directory.

---

### **2. terraform fmt**

**The Rule:** Formats `.tf` files to the **canonical style**. Does not change logic.

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-check` | Check if files are formatted. Returns non-zero exit code if not. **Does not modify files.** |
| `-recursive` | Format files in subdirectories too |
| `-diff` | Show the formatting differences |

**Key Facts:**

- Only formats `.tf` and `.tfvars` files.
- `-check` is perfect for **CI/CD pipelines** (fail build if code is not formatted).
- Does **not** validate syntax or logic. Only whitespace and style.

*Exam Trigger:* "Enforce consistent code style in CI pipeline" → `terraform fmt -check`.

---

### **3. terraform validate**

**The Rule:** Validates configuration for **syntax and internal consistency**. Makes **no API calls** to any provider.

**Key Facts:**

- Checks for syntax errors, invalid references, type mismatches, missing required arguments.
- **Requires `terraform init` first** (needs provider schemas to validate resource arguments).
- Does **not** check if the resources can actually be created (no cloud API calls).
- Does **not** check state. Only checks configuration files.

*Exam Trap:* `validate` requires `init` because it needs provider schemas, but it does **not** access remote state or make API calls.

*Exam Trigger:* "Check syntax without making any infrastructure changes" → `terraform validate`.

---

### **4. terraform plan**

**The Rule:** Creates an **execution plan** showing what Terraform will do. Compares desired state (config) to actual state (state file + API refresh).

**What It Does:**

1. Reads current state.
2. Refreshes state against real infrastructure (API calls).
3. Compares config to refreshed state.
4. Outputs a diff of changes: **create (+), update (~), destroy (-)**.

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-out=FILE` | Save the plan to a binary file for later `apply` |
| `-destroy` | Generate a plan to destroy all resources |
| `-replace=RESOURCE` | Plan to force-replace a specific resource (destroy + recreate) |
| `-target=RESOURCE` | Plan only for a specific resource and its dependencies |
| `-refresh-only` | Only refresh state against real infrastructure, propose no config changes |
| `-var 'KEY=VALUE'` | Set an input variable value |
| `-var-file=FILE` | Load variable values from a file |

**Key Facts:**

- **`-out` is critical for CI/CD.** A saved plan guarantees that `apply` executes **exactly** what was reviewed.
- **`-refresh-only`** is used to detect and reconcile **drift** (real infra changed outside Terraform).
- **`-replace`** is the modern replacement for the deprecated `terraform taint` command.
- **`-target`** should be used for **exceptional situations only**, not normal workflow.

*Exam Trigger:* "Ensure exact changes reviewed in PR are applied" → `terraform plan -out=plan.tfplan`.

*Exam Trap:* `terraform plan` **does** make API calls (to refresh state). Only `validate` makes no API calls.

---

### **5. terraform apply**

**The Rule:** Executes the changes to reach the desired state. By default, generates a plan and asks for confirmation.

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-auto-approve` | Skip the interactive approval prompt |
| `PLANFILE` | Apply a saved plan file (no prompt, no re-planning) |
| `-replace=RESOURCE` | Force-replace a specific resource |
| `-target=RESOURCE` | Apply only a specific resource and its dependencies |

**Key Facts:**

- When you pass a **saved plan file**, Terraform applies **exactly** that plan. No re-computation, no prompt.
- Without a saved plan, `apply` runs a new plan and asks "Do you want to perform these actions?"
- `-auto-approve` is used in **automation/CI/CD** when human approval is handled elsewhere.

**Exam Contrast: apply with vs. without saved plan**

| **Behavior** | **`terraform apply`** | **`terraform apply plan.tfplan`** |
| --- | --- | --- |
| **Generates new plan?** | Yes | No (uses saved plan) |
| **Prompts for approval?** | Yes | No (already reviewed) |
| **Can have drift since plan?** | Yes (re-reads state) | No (locked to saved plan) |

*Exam Trigger:* "Automate apply in pipeline without human prompt" → `terraform apply -auto-approve` or apply a saved plan file.

---

### **6. terraform destroy**

**The Rule:** Destroys **all resources** managed by the current configuration/state.

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-target=RESOURCE` | Destroy only a specific resource |
| `-auto-approve` | Skip confirmation prompt |

**Key Facts:**

- Equivalent to `terraform apply -destroy`.
- Shows a plan of what will be destroyed and asks for confirmation (unless `-auto-approve`).
- Respects resource dependencies (destroys in reverse dependency order).

*Exam Trap:* `terraform destroy` destroys **everything** in state, not just what is in the current `.tf` files. If you removed a resource from config, `apply` would destroy it. But `destroy` destroys **all** resources.

---

### **7. terraform output**

**The Rule:** Reads and displays **output values** from the state file.

**Key Flags:**

| **Flag** | **Purpose** |
| --- | --- |
| `-json` | Output in JSON format |
| `-raw` | Output raw string value (no quotes, no newline) |

**Key Facts:**

- `terraform output` (no args) shows all outputs.
- `terraform output VARIABLE_NAME` shows a specific output.
- `-raw` is useful for piping to other commands (e.g., `terraform output -raw instance_ip | ssh ...`).
- Reads from **state only**. Does not make API calls.

---

### **8. terraform show**

**The Rule:** Displays the **current state** or a **saved plan file** in human-readable format.

**Usage:**

```
terraform show                  # Show current state
terraform show plan.tfplan      # Show saved plan file
terraform show -json            # Output state as JSON
```

**Key Facts:**

- Useful for inspecting what is in state or reviewing a saved plan.
- `-json` flag outputs structured JSON for programmatic consumption.

---

### **9. terraform state (Subcommands)**

**The Rule:** Advanced commands to directly inspect and manipulate the **state file**.

| **Subcommand** | **Purpose** |
| --- | --- |
| `terraform state list` | List all resources tracked in state |
| `terraform state show RESOURCE` | Show detailed attributes of a single resource in state |
| `terraform state mv SOURCE DEST` | Move/rename a resource in state (avoids destroy + recreate) |
| `terraform state rm RESOURCE` | Remove a resource from state (Terraform "forgets" it, resource still exists) |
| `terraform state pull` | Download and print the remote state to stdout |
| `terraform state push` | Upload a local state file to the remote backend |

**Key Facts:**

- `state mv` is used when you **rename** a resource or move it to a **module**. Without it, Terraform would destroy the old and create new.
- `state rm` makes Terraform **forget** a resource. The real infrastructure is **not deleted**. Used when you want to stop managing a resource.
- `state pull` and `state push` are for **advanced** state recovery scenarios.

*Exam Trigger:* "Rename a resource without destroying it" → `terraform state mv`.

*Exam Trigger:* "Stop managing a resource but keep it running" → `terraform state rm`.

*Exam Trap:* `terraform state rm` does **not** destroy the resource. It only removes it from state.

---

### **10. terraform console**

**The Rule:** Interactive console for evaluating **Terraform expressions**.

**Key Facts:**

- Opens a REPL where you can test interpolations, functions, and variable references.
- Reads from current state and configuration.
- Useful for debugging expressions before putting them in config.

```
$ terraform console
> max(5, 12, 9)
12
> format("Hello, %s!", "Terraform")
"Hello, Terraform!"
```

---

### **11. terraform graph**

**The Rule:** Outputs a visual **dependency graph** of resources in **DOT format**.

**Key Facts:**

- Output is in DOT format (used by Graphviz).
- Pipe to Graphviz to generate an image: `terraform graph | dot -Tpng > graph.png`.
- Shows how resources depend on each other.

*Exam Trigger:* "Visualize resource dependencies" → `terraform graph`.

---

### **12. terraform workspace**

**The Rule:** Manage **multiple state files** within a single configuration (e.g., dev, staging, prod).

| **Subcommand** | **Purpose** |
| --- | --- |
| `terraform workspace new NAME` | Create a new workspace |
| `terraform workspace select NAME` | Switch to a workspace |
| `terraform workspace list` | List all workspaces |
| `terraform workspace delete NAME` | Delete a workspace |
| `terraform workspace show` | Show current workspace name |

**Key Facts:**

- Default workspace is called `default`. It cannot be deleted.
- Each workspace has its **own state file**.
- Use `terraform.workspace` in config to reference the current workspace name.
- Workspaces are **not** a full environment isolation solution (HCP Terraform/Terraform Cloud workspaces are different and more robust).

*Exam Trap:* CLI workspaces share the **same configuration and backend**. They only separate **state**. They are not the same as HCP Terraform workspaces.

---

### **13. terraform taint (DEPRECATED)**

**The Rule:** `terraform taint` marked a resource for **forced replacement** on the next apply. **Deprecated** in favor of the `-replace` flag.

**Old Way (Deprecated):**

```
terraform taint aws_instance.web
terraform apply
```

**New Way (Current):**

```
terraform apply -replace="aws_instance.web"
```

**Why Deprecated:**

- `taint` modified state directly (side effect between plan and apply).
- `-replace` is declared in the plan/apply command itself (cleaner, auditable).

*Exam Trigger:* "Force recreate a resource" → `terraform apply -replace="RESOURCE"`.

*Exam Trap:* `terraform taint` still works but is **deprecated**. The exam prefers `-replace`.

---

### **14. terraform import**

**The Rule:** Bring an **existing resource** (created outside Terraform) into Terraform state.

**CLI Method (Legacy, Still Works):**

```
terraform import aws_instance.web i-1234567890abcdef0
```

**Import Block Method (Preferred in 004):**

```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}
```

**Key Facts:**

- CLI `import` writes directly to state. You must **manually write** the matching config.
- The `import` block can be combined with `terraform plan` to **preview** the import and can **generate configuration**.
- The `import` block is declarative and auditable (lives in code, goes through normal plan/apply).

*Exam Trigger:* "Import existing resource with plan preview" → Use `import` block.

*Exam Trap:* `terraform import` (CLI) does **not** generate configuration. You must write the resource block yourself. The `import` block with plan **can** generate config.

---

### **15. terraform get**

**The Rule:** Downloads and updates **modules only**. Does **not** initialize backends or download providers.

**Key Facts:**

- `terraform init` also downloads modules (it does everything `get` does, plus more).
- `terraform get` is a subset of `init`. Useful if you only changed module sources and want a quick update.
- Rarely used in practice since `init` covers it.

*Exam Trigger:* "What command downloads modules?" → Both `terraform init` and `terraform get`. But `init` also handles providers and backends.

---

### **SECTION 3: CI/CD PATTERNS**

*Automation best practices the exam tests.*

### **1. The Saved Plan Pattern**

**The Rule:** In CI/CD, always use a **saved plan** to guarantee what was reviewed is what gets applied.

**Pattern:**

```
# Step 1: PR is opened → CI runs plan and saves it
terraform plan -out=plan.tfplan

# Step 2: Team reviews the plan output in the PR

# Step 3: PR is merged → CD applies the saved plan
terraform apply plan.tfplan
```

**Why This Matters:**

- Without a saved plan, `apply` generates a **new** plan at apply time. Infrastructure may have changed between review and apply.
- A saved plan is a **binary file** that locks in the exact changes.
- When applying a saved plan, Terraform does **not** prompt for approval (the review already happened).

*Exam Trigger:* "Ensure reviewed changes are exactly what gets applied" → Saved plan pattern.

---

### **2. The -target Flag (Exceptional Use)**

**The Rule:** `-target` limits operations to a specific resource. It is for **exceptional use only**.

**When to Use:**

- Recovering from a partially failed apply.
- Working around a bug in one resource while others are fine.
- Initial development iteration on a single resource.

**When NOT to Use:**

- Normal day-to-day workflow.
- As a substitute for proper resource organization (use modules instead).

*Exam Trap:* `-target` is **not** a recommended part of normal workflow. The exam expects you to know it exists but also know it should be used sparingly.

---

### **3. validate vs. plan**

| **Aspect** | **`terraform validate`** | **`terraform plan`** |
| --- | --- | --- |
| **API Calls** | **None** | **Yes** (refreshes state against real infra) |
| **Checks** | Syntax, types, references, internal consistency | Everything validate checks + real-world feasibility |
| **State Access** | **No** | **Yes** |
| **Requires Init** | **Yes** (needs provider schemas) | **Yes** |
| **Speed** | **Fast** (no network calls) | **Slower** (calls cloud APIs) |
| **Use In CI** | Early gate (fail fast on syntax errors) | Full pre-apply check |

*Exam Trigger:* "Check configuration without accessing remote state or cloud APIs" → `terraform validate`.

---

### **4. terraform get vs. terraform init**

| **Aspect** | **`terraform get`** | **`terraform init`** |
| --- | --- | --- |
| **Downloads Modules** | Yes | Yes |
| **Downloads Providers** | **No** | Yes |
| **Initializes Backend** | **No** | Yes |
| **Creates Lock File** | **No** | Yes |
| **Must Run First** | No (init must run first) | **Yes** (required before any command) |

*Exam Trigger:* "What is the difference between get and init?" → `get` only handles modules. `init` handles modules + providers + backend + lock file.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **Core workflow order?** → Write → Init → Plan → Apply (→ Destroy).
2. **First command for any new project?** → `terraform init`.
3. **Format code in CI without modifying?** → `terraform fmt -check`.
4. **Validate syntax without API calls?** → `terraform validate`.
5. **Save plan for CI/CD?** → `terraform plan -out=plan.tfplan`.
6. **Apply saved plan?** → `terraform apply plan.tfplan` (no prompt, no re-plan).
7. **Skip approval prompt in automation?** → `terraform apply -auto-approve`.
8. **Force-replace a resource?** → `terraform apply -replace="RESOURCE"` (not `taint`).
9. **Rename a resource without destroying it?** → `terraform state mv`.
10. **Stop managing a resource but keep it alive?** → `terraform state rm`.
11. **Import existing infrastructure?** → `import` block (preferred) or `terraform import` (CLI).
12. **Detect drift only?** → `terraform plan -refresh-only`.
13. **Visualize dependencies?** → `terraform graph` (DOT format).
14. **Only download modules?** → `terraform get`. But `init` does this too.
15. **`taint` is deprecated?** → Yes. Use `-replace` flag instead.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "CI/CD Pipeline Safety" (Saved Plan)**

**The Situation:** A DevOps team uses Terraform in a CI/CD pipeline. During a pull request, `terraform plan` runs and the team reviews the output. When the PR is merged, `terraform apply` runs automatically. However, between the PR review and the merge, another engineer manually changed a security group in the AWS console. The apply creates unexpected changes that were not reviewed.

**The Options:**

A. Run `terraform plan` again before apply to catch the drift.

B. Use `terraform plan -out=plan.tfplan` during PR, then `terraform apply plan.tfplan` on merge.

C. Use `terraform apply -auto-approve` to skip the confirmation.

D. Use `terraform validate` before apply to verify the changes.

**The Logic:**

- **Trap:** Running plan again (Option A) catches drift but still produces a **new** plan that was not reviewed by the team.
- **Trap:** `-auto-approve` (Option C) skips prompts but still generates a new plan at apply time (same drift problem).
- **Trap:** `validate` (Option D) only checks syntax. It does not detect infrastructure drift.
- **The Fix:** **Option B**. A **saved plan file** locks in the exact changes at review time. When `apply plan.tfplan` runs, it executes **exactly** what was reviewed. If state has drifted, Terraform will refuse to apply the stale plan and error out, forcing a re-plan.

---

### **Scenario 2: The "Force Recreate" (Replace Flag)**

**The Situation:** An EC2 instance managed by Terraform is experiencing application issues. The team suspects the instance's root volume is corrupted. They want to force Terraform to **destroy and recreate** the instance without changing any configuration.

**The Options:**

A. Run `terraform destroy -target=aws_instance.web` then `terraform apply`.

B. Run `terraform taint aws_instance.web` then `terraform apply`.

C. Run `terraform apply -replace="aws_instance.web"`.

D. Run `terraform state rm aws_instance.web` then `terraform apply`.

**The Logic:**

- **Trap:** Destroy + apply (Option A) works but is two steps and risky (if apply fails, you have no instance).
- **Trap:** `taint` (Option B) still works but is **deprecated**. The exam prefers the modern approach.
- **Trap:** `state rm` (Option D) makes Terraform forget the instance. Apply would create a **new** instance but the old one still exists (orphaned resource).
- **The Fix:** **Option C**. `terraform apply -replace` is the **current recommended method**. It plans a destroy + create of that specific resource in a single atomic operation. No deprecated commands, no orphaned resources.

---

### **Scenario 3: The "Syntax Check Gate" (validate vs. plan)**

**The Situation:** A platform team wants to add a fast validation step to their CI pipeline that runs on every commit. The step should catch syntax errors, invalid references, and type mismatches in Terraform configuration. It should **not** require cloud credentials or make any API calls, because the CI runner does not have access to the AWS account.

**The Options:**

A. Run `terraform plan` with read-only credentials.

B. Run `terraform validate`.

C. Run `terraform fmt -check`.

D. Run `terraform apply -refresh-only`.

**The Logic:**

- **Trap:** `plan` (Option A) requires API calls and credentials to refresh state. The CI runner has no credentials.
- **Trap:** `fmt -check` (Option C) only checks formatting/style. It does not catch invalid resource arguments or broken references.
- **Trap:** `-refresh-only` (Option D) requires full credentials and only checks drift, not syntax.
- **The Fix:** **Option B**. `terraform validate` checks syntax, types, and references using only the **provider schemas** downloaded during `init`. It makes **zero API calls** and requires no cloud credentials. Perfect for a fast CI gate.

---

### **Scenario 4: The "Forgotten Resource" (state rm)**

**The Situation:** A team has been managing a legacy database with Terraform. The DBA team now wants to manage this database manually and does not want Terraform to touch it anymore. However, the database must **remain running** and must not be destroyed.

**The Options:**

A. Remove the resource block from the `.tf` file and run `terraform apply`.

B. Run `terraform destroy -target=aws_db_instance.legacy`.

C. Run `terraform state rm aws_db_instance.legacy` and remove the resource block from config.

D. Comment out the resource block in the `.tf` file.

**The Logic:**

- **Trap:** Removing the block and running apply (Option A) causes Terraform to **destroy** the database (it sees a resource in state with no matching config).
- **Trap:** `destroy -target` (Option B) explicitly destroys the database. The DBA team wants it running.
- **Trap:** Commenting out (Option D) has the same effect as removing — Terraform will try to destroy on next apply.
- **The Fix:** **Option C**. `terraform state rm` removes the database from Terraform's state. Terraform "forgets" it exists. The real database **keeps running**. Then remove the resource block from config so Terraform does not try to recreate it.

---

### **Scenario 5: The "Backend Migration" (init flags)**

**The Situation:** A team has been using a local backend for Terraform state. They now want to migrate to an S3 backend for remote state storage. They have updated the backend configuration in the `.tf` files to point to an S3 bucket. They need to move the existing state to the new backend without losing any tracked resources.

**The Options:**

A. Run `terraform init`.

B. Run `terraform init -reconfigure`.

C. Run `terraform init -migrate-state`.

D. Run `terraform state push` to upload state to S3.

**The Logic:**

- **Trap:** Plain `terraform init` (Option A) may detect the backend change and prompt, but the correct explicit flag ensures migration.
- **Trap:** `-reconfigure` (Option B) reconfigures the backend but **does not migrate** existing state. The new backend starts with empty state (all resources appear "new").
- **Trap:** `state push` (Option D) is a low-level command that could overwrite remote state and does not update backend configuration.
- **The Fix:** **Option C**. `terraform init -migrate-state` detects the backend change and **copies** the existing local state to the new S3 backend. All tracked resources are preserved. This is the safe, supported migration path.
