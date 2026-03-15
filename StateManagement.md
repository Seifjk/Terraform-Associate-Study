# State Management & Maintain Infrastructure

### **SECTION 1: STATE OPERATIONS (CLI)**

*Core commands for inspecting and modifying Terraform state.*

### **1. Listing and Inspecting State**

- **The Rule:** Terraform tracks every managed resource in a **state file** (`terraform.tfstate`). CLI commands let you query it directly.

**terraform state list:**

- Lists all resource addresses currently tracked in state.

```bash
terraform state list
# aws_instance.web
# aws_s3_bucket.data
# module.vpc.aws_vpc.main
```

**terraform state show:**

- Shows all attributes of a specific resource in state.

```bash
terraform state show aws_instance.web
# id          = "i-0abc123def456"
# ami         = "ami-0c55b159cbfafe1f0"
# instance_type = "t3.micro"
# private_ip  = "10.0.1.50"
```

---

### **2. Moving and Renaming Resources**

**terraform state mv:**

- **The Rule:** Moves or renames a resource in state **without modifying real infrastructure**.
- Terraform updates the state file so the resource maps to a new address.

```bash
# Rename a resource
terraform state mv aws_instance.old_name aws_instance.new_name

# Move a resource into a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Move a resource out of a module
terraform state mv module.compute.aws_instance.web aws_instance.web
```

**Use Cases:**

- Refactoring code (renaming resources, reorganizing into modules).
- Without `state mv`, Terraform would **destroy** the old name and **create** the new name.

*Exam Trigger:* "Rename a resource without destroying infrastructure" --> `terraform state mv`.

---

### **3. Removing Resources from State**

**terraform state rm:**

- **The Rule:** Removes a resource from state. The **real resource continues to exist** -- Terraform just stops managing it.

```bash
terraform state rm aws_instance.legacy
# Removed aws_instance.legacy
# The EC2 instance still runs in AWS, Terraform no longer tracks it
```

**Use Cases:**

- Hand off a resource to another Terraform workspace or team.
- Stop managing a resource that was imported by mistake.
- Transition a resource to manual management.

*Exam Trap:* `state rm` does **NOT** destroy infrastructure. It only removes the record from state.

---

### **4. Pulling and Pushing State**

**terraform state pull:**

- Downloads current remote state to **stdout** (JSON).
- Safe read-only operation.

```bash
terraform state pull > backup.tfstate
```

**terraform state push:**

- Uploads a local state file to the remote backend.
- **Dangerous** -- can overwrite remote state. Use only for disaster recovery.

```bash
terraform state push backup.tfstate
```

*Exam Trap:* `state push` is rarely the right answer. It bypasses locking and can corrupt state if used carelessly.

---

### **SECTION 2: DECLARATIVE STATE MANAGEMENT (NEW IN 004 -- HEAVILY TESTED)**

*Config-driven replacements for CLI state commands. Reviewable in pull requests.*

### **1. The moved Block (Replaces state mv)**

- **The Rule:** Declare resource renames and module moves **in configuration**. Terraform handles the state update during `apply`.

```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```

**Key Facts:**

- **Declarative:** Lives in `.tf` files, reviewable in PRs.
- **Auditable:** Team can see exactly what was renamed and when.
- **Module Moves:** Can move resources into or out of modules.

```hcl
# Move resource into a module
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

- **Cleanup:** Remove the `moved` block after a successful apply (it has served its purpose).
- **Plan Output:** `terraform plan` shows the move, not a destroy/create.

*Exam Trigger:* "Rename resource in a way that is reviewable in version control" --> `moved` block.

---

### **2. The removed Block (Replaces state rm)**

- **The Rule:** Declare that Terraform should stop managing a resource **without destroying it**.

```hcl
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false  # Keep the real resource, just stop managing
  }
}
```

**Key Facts:**

- `destroy = false` --> Resource stays in the real world, Terraform forgets it (same as `state rm`).
- `destroy = true` (default) --> Terraform **actually destroys** the resource.
- **Declarative:** Reviewable in PRs, auditable in version control.
- **Cleanup:** Remove the `removed` block after a successful apply.

*Exam Trap:* If you omit the `lifecycle` block or set `destroy = true`, the resource gets **destroyed** -- not just unmanaged.

---

### **3. The import Block (Replaces terraform import CLI)**

- **The Rule:** Bring existing infrastructure under Terraform management using configuration, not CLI.

```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}
```

**Key Facts:**

- Part of the normal `plan`/`apply` workflow.
- **Generate Config:** Terraform can auto-generate the matching resource block.

```bash
terraform plan -generate-config-out=generated.tf
```

- **Reviewable:** Import is visible in `terraform plan` output before apply.
- **Cleanup:** Remove the `import` block after a successful apply.

*Exam Trigger:* "Import existing infrastructure in a reviewable workflow" --> `import` block.

---

### **4. CLI vs. Declarative Comparison**

| **Operation** | **CLI Command** | **Declarative Block** | **Key Difference** |
| --- | --- | --- | --- |
| **Rename / Move** | `terraform state mv` | `moved { }` | `moved` is reviewable in PRs |
| **Stop Managing** | `terraform state rm` | `removed { }` | `removed` is reviewable in PRs |
| **Import Existing** | `terraform import` | `import { }` | `import` block is part of plan/apply |

**The Rule:** The 004 exam **strongly prefers** declarative blocks over CLI commands. CLI commands are imperative, one-off, and not tracked in version control. Declarative blocks are auditable, reviewable, and follow the Terraform workflow.

---

### **SECTION 3: DRIFT DETECTION AND RESOURCE MANAGEMENT**

*Objective 7 -- Detecting and correcting out-of-band changes.*

### **1. What Is Drift?**

- **The Rule:** Drift occurs when someone changes infrastructure **outside of Terraform** (e.g., manual console change, another tool, API call).
- Terraform state says one thing; real infrastructure says another.

---

### **2. Detecting Drift**

**terraform plan:**

- **The Rule:** `terraform plan` compares **state** to **real infrastructure** and shows differences.
- If someone manually changed an instance type from `t3.micro` to `t3.large`, `plan` shows the diff.

```bash
terraform plan
# ~ aws_instance.web
#     instance_type: "t3.large" -> "t3.micro"
#     # Terraform will change it back to match config
```

---

### **3. Correcting Drift**

**terraform apply:**

- **The Rule:** `terraform apply` brings real infrastructure **back to match your configuration**.
- If someone manually changed something, `apply` reverts it.

**terraform apply -refresh-only:**

- **The Rule:** Updates state to match reality **WITHOUT changing infrastructure**.
- Use when the manual change was intentional and you want Terraform to accept it.

```bash
terraform apply -refresh-only
# Updates state to reflect current real-world values
# Does NOT modify any infrastructure
```

*Exam Trigger:* "Update state to match manual changes without modifying infrastructure" --> `terraform apply -refresh-only`.

---

### **4. Replacing Resources**

**terraform apply -replace:**

- **The Rule:** Forces Terraform to **destroy and recreate** a specific resource.
- Preferred over the deprecated `terraform taint` command.

```bash
terraform apply -replace=aws_instance.web
# Destroys and recreates aws_instance.web
```

**Use Cases:**

- Resource is in a bad state (corrupted, misconfigured at OS level).
- Force fresh provisioning.

*Exam Trap:* `terraform taint` is **deprecated**. The correct answer is `terraform apply -replace`.

---

### **5. terraform refresh**

- **The Rule:** Standalone command that updates state to match real infrastructure.
- **Discouraged** -- use `terraform apply -refresh-only` instead.
- Refresh is already **implicit** in every `plan` and `apply`.

---

### **SECTION 4: CLI WORKSPACES**

*Isolated state files for managing multiple environments.*

### **1. Workspace Commands**

```bash
terraform workspace new dev        # Create new workspace
terraform workspace select dev     # Switch to workspace
terraform workspace list           # List all workspaces
terraform workspace show           # Show current workspace
terraform workspace delete dev     # Delete workspace (must not be current)
```

---

### **2. How Workspaces Store State**

- **The Rule:** Each workspace has its **own state file**.
- **Default workspace:** Uses `terraform.tfstate` in the root.
- **Named workspaces:** Uses `terraform.tfstate.d/<workspace>/terraform.tfstate`.

```
project/
  terraform.tfstate              # default workspace
  terraform.tfstate.d/
    dev/
      terraform.tfstate          # dev workspace
    staging/
      terraform.tfstate          # staging workspace
```

---

### **3. The terraform.workspace Variable**

- **The Rule:** Use `terraform.workspace` to reference the current workspace name in configuration.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Environment = terraform.workspace
  }
}
```

---

### **4. Workspace Rules**

- The **default** workspace always exists and **cannot be deleted**.
- You cannot delete the workspace you are currently in.
- Workspaces share the same configuration but have **separate state**.

**Exam Trap:** CLI workspaces are **NOT** the same as HCP Terraform (Terraform Cloud) workspaces. CLI workspaces share the same backend config and code. HCP Terraform workspaces can have entirely different configurations, variables, and access controls.

---

### **SECTION 5: DEBUGGING AND TROUBLESHOOTING**

*Environment variables and tools for diagnosing issues.*

### **1. TF_LOG Environment Variable**

- **The Rule:** Set `TF_LOG` to enable detailed logging. Levels from most to least verbose:

| **Level** | **Detail** |
| --- | --- |
| **TRACE** | Most verbose -- all internal operations |
| **DEBUG** | Detailed debugging information |
| **INFO** | General operational messages |
| **WARN** | Warnings only |
| **ERROR** | Errors only |

```bash
export TF_LOG=TRACE
terraform plan
```

*Exam Trigger:* "Most verbose logging" --> `TF_LOG=TRACE`.

---

### **2. Targeted Logging**

- **TF_LOG_CORE:** Logs for Terraform core only (excludes provider operations).
- **TF_LOG_PROVIDER:** Logs for provider plugins only (excludes core operations).

```bash
export TF_LOG_CORE=TRACE
export TF_LOG_PROVIDER=DEBUG
```

---

### **3. Log File Output**

- **TF_LOG_PATH:** Write logs to a file instead of stderr.

```bash
export TF_LOG=TRACE
export TF_LOG_PATH="./terraform.log"
terraform plan
# Logs written to ./terraform.log
```

---

### **4. terraform console**

- **The Rule:** Interactive REPL for testing expressions and exploring state.

```bash
terraform console
> length(var.subnets)
3
> aws_instance.web.private_ip
"10.0.1.50"
> terraform.workspace
"dev"
```

*Exam Trigger:* "Test Terraform expressions interactively" --> `terraform console`.

---

### **SECTION 6: SENSITIVE DATA IN STATE**

*Protecting secrets that end up in the state file.*

### **1. The sensitive Attribute**

- **The Rule:** Marking a variable or output as `sensitive = true` hides the value from CLI output, but it is **still stored in plain text in the state file**.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "password" {
  value     = var.db_password
  sensitive = true
}
```

```bash
terraform output password
# (sensitive value)
```

*Exam Trap:* `sensitive = true` hides from **output/plan display** only. The value is still **in the state file in plain text**.

---

### **2. Ephemeral Values (New in 004)**

- **The Rule:** Ephemeral values are **never written to state**. They exist only during the plan/apply operation.
- Use for credentials, tokens, and secrets that must not persist.

*Exam Trigger:* "Value must never appear in state file" --> Ephemeral values.

---

### **3. Securing State**

**The Rule:** Treat the state file as **sensitive data**. Protect it with:

- **Encryption at rest:** Use a backend that encrypts (S3 with SSE, HCP Terraform).
- **Encryption in transit:** Use TLS (HTTPS) for remote backends.
- **Access control:** Restrict who can read/write state (IAM policies, workspace permissions).
- **State locking:** Prevent concurrent writes (DynamoDB for S3, built-in for HCP Terraform).
- **Never commit `terraform.tfstate` to version control.** Add it to `.gitignore`.

*Exam Trap:* The state file contains **all resource attributes including secrets**. Even if you mark values `sensitive`, they are in state. Secure the backend.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **List resources in state?** --> `terraform state list`.
2. **Show resource attributes?** --> `terraform state show <resource>`.
3. **Rename resource without destroying?** --> `moved` block (preferred) or `terraform state mv`.
4. **Stop managing a resource without destroying it?** --> `removed` block with `destroy = false` (preferred) or `terraform state rm`.
5. **Import existing infrastructure (reviewable)?** --> `import` block with `terraform plan -generate-config-out`.
6. **CLI vs. declarative?** --> Exam prefers declarative blocks (`moved`, `removed`, `import`) -- auditable, reviewable.
7. **Detect drift?** --> `terraform plan`.
8. **Correct drift?** --> `terraform apply`.
9. **Accept manual changes into state?** --> `terraform apply -refresh-only`.
10. **Force recreate a resource?** --> `terraform apply -replace` (not `taint`).
11. **Most verbose logging?** --> `TF_LOG=TRACE`.
12. **Logs for providers only?** --> `TF_LOG_PROVIDER`.
13. **Test expressions interactively?** --> `terraform console`.
14. **Sensitive but still in state?** --> `sensitive = true`.
15. **Never written to state?** --> Ephemeral values.
16. **CLI workspaces = HCP Terraform workspaces?** --> **NO.** They are completely different concepts.
17. **Default workspace?** --> Always exists, cannot be deleted.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "Refactored Module" (moved block)**

**The Situation:** Your team refactored Terraform code and moved `aws_instance.web` into a new module called `compute`. Without intervention, `terraform plan` shows the original resource will be **destroyed** and a new one **created** inside the module. You need to avoid any downtime or resource recreation.

**The Options:**

A. Run `terraform apply` and let Terraform destroy and recreate the instance.

B. Add a `moved` block mapping `aws_instance.web` to `module.compute.aws_instance.web`.

C. Manually edit the state file JSON to update the resource address.

D. Run `terraform import module.compute.aws_instance.web <instance-id>` after deleting the old resource from state.

**The Logic:**

- **Trap:** Option A destroys production infrastructure unnecessarily.
- **Trap:** Option C is error-prone, unsupported, and can corrupt state.
- **Trap:** Option D works but requires two manual steps and is not reviewable.
- **The Fix:** **Option B**. The `moved` block declaratively tells Terraform the resource was renamed. `terraform plan` shows a move, not a destroy/create. It is reviewable in a PR and requires no manual state manipulation.

---

### **Scenario 2: The "Legacy Hand-Off" (removed block)**

**The Situation:** Your team created an RDS database with Terraform. Another team now wants to manage it with their own Terraform workspace. You need to **stop managing** the database without destroying it. The change must be reviewable and auditable.

**The Options:**

A. Run `terraform destroy -target=aws_db_instance.legacy`.

B. Run `terraform state rm aws_db_instance.legacy`.

C. Add a `removed` block with `destroy = false` for the database.

D. Delete the resource block from configuration and run `terraform apply`.

**The Logic:**

- **Trap:** Option A **destroys** the database -- the opposite of the requirement.
- **Trap:** Option B works but is imperative, not tracked in version control, and not reviewable.
- **Trap:** Option D causes Terraform to **destroy** the resource (it sees it in state but not in config).
- **The Fix:** **Option C**. The `removed` block with `destroy = false` tells Terraform to release the resource from state without destroying it. It is declarative, reviewable in a PR, and the other team can then import it into their workspace.

---

### **Scenario 3: The "Drift Correction" (refresh-only)**

**The Situation:** A developer manually changed the instance type of an EC2 instance from `t3.micro` to `t3.large` through the AWS Console to handle a temporary traffic spike. The spike is over, but the team wants to **keep** the `t3.large` instance type and update Terraform state to reflect reality. They do **not** want Terraform to revert the change.

**The Options:**

A. Run `terraform apply` to bring infrastructure back to the `t3.micro` in configuration.

B. Run `terraform apply -refresh-only` to update state, then update the configuration to `t3.large`.

C. Run `terraform state rm` on the instance and re-import it.

D. Delete the state file and run `terraform init`.

**The Logic:**

- **Trap:** Option A **reverts** the change back to `t3.micro` -- the opposite of the requirement.
- **Trap:** Option C is overly complex and loses other state attributes.
- **Trap:** Option D destroys all state knowledge and risks orphaning resources.
- **The Fix:** **Option B**. `terraform apply -refresh-only` updates the state to match real infrastructure (`t3.large`) without changing anything. Then update the config to `t3.large` so future plans show no diff.

---

### **Scenario 4: The "Import Existing Infrastructure" (import block)**

**The Situation:** Your company has an S3 bucket (`my-app-data`) that was created manually through the AWS Console months ago. The team wants to bring it under Terraform management. The import must be reviewable in a pull request and follow the standard plan/apply workflow.

**The Options:**

A. Run `terraform import aws_s3_bucket.data my-app-data` from the CLI.

B. Add an `import` block in configuration and run `terraform plan -generate-config-out=generated.tf`.

C. Write the resource block manually and run `terraform apply` (Terraform will detect the existing bucket).

D. Delete the bucket and let Terraform recreate it.

**The Logic:**

- **Trap:** Option A works but is imperative and not reviewable in a PR.
- **Trap:** Option C fails -- Terraform tries to **create** the bucket and gets a "bucket already exists" error.
- **Trap:** Option D causes data loss -- the bucket has production data.
- **The Fix:** **Option B**. The `import` block declares the import in configuration. Running `terraform plan -generate-config-out=generated.tf` auto-generates the matching resource block. The entire workflow is part of plan/apply and reviewable in a pull request.

---

### **Scenario 5: The "Workspace Confusion" (CLI Workspaces)**

**The Situation:** A team uses CLI workspaces (`dev`, `staging`, `prod`) to manage three environments. A new engineer switches to the `dev` workspace and runs `terraform destroy`, expecting it to only affect dev resources. However, they notice the `dev` workspace state is empty and suspect they may be in the wrong workspace. They want to verify which workspace they are in and see all available workspaces.

**The Options:**

A. Run `terraform workspace list` and `terraform workspace show`.

B. Check the `terraform.tfstate` file in the root directory.

C. Run `terraform state list` and assume the environment from the resource tags.

D. Run `terraform plan` and infer the workspace from the output.

**The Logic:**

- **Trap:** Option B shows the **default** workspace state, not necessarily the current one.
- **Trap:** Option C shows resources but does not confirm the workspace name.
- **Trap:** Option D shows planned changes but does not explicitly name the workspace.
- **The Fix:** **Option A**. `terraform workspace show` outputs the current workspace name. `terraform workspace list` shows all workspaces with an asterisk next to the active one. These are the definitive commands for workspace identification.
